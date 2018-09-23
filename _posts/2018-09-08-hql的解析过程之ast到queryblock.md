---
layout:     post
title:      hive编译过程
subtitle:   hive源码学习之-ast到queryblock
date:       2018-09-08
author:     鲁湘宁
header-img: img/doc1-bg-alone.jpg
catalog: true
tags:
    - hive 入门 3.1.0 源码 hql解析 hql ast queryblock
---
## 前言
AST Tree仍然非常复杂，不够结构化，不方便直接翻译为MapReduce程序，AST Tree转化为QueryBlock就是将SQL进一部抽象和结构化。  
QueryBlock是一条SQL最基本的组成单元，包括三个部分：输入源，计算过程，输出。简单来讲一个QueryBlock就是一个子查询。  

## 生成QueryBlock

### 总体流程
ast是一个描述sql对应的结构的描述体。ast更多的是反映了该sql对应了哪些流程结构，而实际上一个结构及其子结构才对应一个具体的查询流程，而且ast对于一个查询的具体元数据信息是缺失的。在生成map-reduce之前，需要一个描述具体map-reduce信息的结构体，即QueryBlock。  
一个map-reduce流程的主要组成是输入、计算过程和输出，所以QueryBlock也就相应的对这三方面的信息进行了描述。

### 生成过程
QueryBlock的生成是一个递归调用的过程。每一个ast中的QUERY\_TOK对应一个QueryBlock。对于一个QueryBlock中的描述信息，会通过doPhase1进行递归调用来进行生成，也会在后续查询meta信息把元数据信息补充进去。  
一个qb结构如下   
![](https://luxiangning.github.io/img/hive/hive-qb_01.png)  
具体各个字段的含义为:  
numJoins: join的个数，3.1.0没有使用  
numGbys: groupby的个数，3.1.0没有使用  
numSels: select的个数，3.1.0没有使用  
numSelDi: select distinct的个数，3.1.0没有使用  
aliasToTabs: 记录了当前的查询中别名与表的关系，如select * from report r,则aliasToTabs会存储 r->report映射关系。因为在进行相关join等操作时，很多都是用别名进行的操作，所以这里别名作为key也是应有之义。   
aliasToSubq: 记录了子查询别名与子查询的对应关心。  
例子如图所示  
![](https://luxiangning.github.io/img/hive/hive-qb_02.png) 
aliases: 上述两者的合集  
qbp: 当前qb的parse信息,如下图所示    
![](https://luxiangning.github.io/img/hive/hive-qb_04.jpg)  
可以看到，qbp中记录了一个QUERY_TOK下对应的所有的相关的子TOK信息。  
qbm: 当前qb的元数据信息，包括分区信息、文件信息等  

展开整个qb如下图所示：  
![](https://luxiangning.github.io/img/hive/hive-qb_03.jpg) 

主要的函数过程如下图所示  
```java  
public boolean doPhase1(ASTNode ast, QB qb, Phase1Ctx ctx_1, PlannerContext plannerCtx)
      throws SemanticException {

    boolean phase1Result = true;
    // 对于每一个QueryBlock,qbp记录了该block对应的相应解析信息
    QBParseInfo qbp = qb.getParseInfo();
    // 跳过递归
    boolean skipRecursion = false;

    if (ast.getToken() != null) {
      skipRecursion = true;
      switch (ast.getToken().getType()) {
      case HiveParser.TOK_SELECTDI:
        qb.countSelDi();
        // fall through
      case HiveParser.TOK_SELECT:
        qb.countSel();
        qbp.setSelExprForClause(ctx_1.dest, ast);

        int posn = 0;
        if (((ASTNode) ast.getChild(0)).getToken().getType() == HiveParser.QUERY_HINT) {
          ParseDriver pd = new ParseDriver();
          String queryHintStr = ast.getChild(0).getText();
          if (LOG.isDebugEnabled()) {
            LOG.debug("QUERY HINT: "+queryHintStr);
          }
          try {
            ASTNode hintNode = pd.parseHint(queryHintStr);
            qbp.setHints(hintNode);
            posn++;
          } catch (ParseException e) {
            throw new SemanticException("failed to parse query hint: "+e.getMessage(), e);
          }
        }

        if ((ast.getChild(posn).getChild(0).getType() == HiveParser.TOK_TRANSFORM)) {
          queryProperties.setUsesScript(true);
        }

        LinkedHashMap<String, ASTNode> aggregations = doPhase1GetAggregationsFromSelect(ast,
            qb, ctx_1.dest);
        doPhase1GetColumnAliasesFromSelect(ast, qbp, ctx_1.dest);
        qbp.setAggregationExprsForClause(ctx_1.dest, aggregations);
        qbp.setDistinctFuncExprsForClause(ctx_1.dest,
            doPhase1GetDistinctFuncExprs(aggregations));
        break;

      case HiveParser.TOK_WHERE:
        qbp.setWhrExprForClause(ctx_1.dest, ast);
        if (!SubQueryUtils.findSubQueries((ASTNode) ast.getChild(0)).isEmpty()) {
          queryProperties.setFilterWithSubQuery(true);
        }
        break;

      case HiveParser.TOK_INSERT_INTO:
        String currentDatabase = SessionState.get().getCurrentDatabase();
        String tab_name = getUnescapedName((ASTNode) ast.getChild(0).getChild(0), currentDatabase);
        qbp.addInsertIntoTable(tab_name, ast);

      case HiveParser.TOK_DESTINATION:
        ctx_1.dest = this.ctx.getDestNamePrefix(ast, qb).toString() + ctx_1.nextNum;
        ctx_1.nextNum++;
        boolean isTmpFileDest = false;
        if (ast.getChildCount() > 0 && ast.getChild(0) instanceof ASTNode) {
          ASTNode ch = (ASTNode) ast.getChild(0);
          if (ch.getToken().getType() == HiveParser.TOK_DIR && ch.getChildCount() > 0
              && ch.getChild(0) instanceof ASTNode) {
            ch = (ASTNode) ch.getChild(0);
            isTmpFileDest = ch.getToken().getType() == HiveParser.TOK_TMP_FILE;
          } else {
            if (ast.getToken().getType() == HiveParser.TOK_DESTINATION
                && ast.getChild(0).getType() == HiveParser.TOK_TAB) {
              String fullTableName = getUnescapedName((ASTNode) ast.getChild(0).getChild(0),
                  SessionState.get().getCurrentDatabase());
              qbp.getInsertOverwriteTables().put(fullTableName.toLowerCase(), ast);
              qbp.setDestToOpType(ctx_1.dest, true);
            }
          }
        }

        // is there a insert in the subquery
        if (qbp.getIsSubQ() && !isTmpFileDest) {
          throw new SemanticException(ErrorMsg.NO_INSERT_INSUBQUERY.getMsg(ast));
        }

        qbp.setDestForClause(ctx_1.dest, (ASTNode) ast.getChild(0));
        handleInsertStatementSpecPhase1(ast, qbp, ctx_1);

        if (qbp.getClauseNamesForDest().size() == 2) {
          // From the moment that we have two destination clauses,
          // we know that this is a multi-insert query.
          // Thus, set property to right value.
          // Using qbp.getClauseNamesForDest().size() >= 2 would be
          // equivalent, but we use == to avoid setting the property
          // multiple times
          queryProperties.setMultiDestQuery(true);
        }

        if (plannerCtx != null && !queryProperties.hasMultiDestQuery()) {
          plannerCtx.setInsertToken(ast, isTmpFileDest);
        } else if (plannerCtx != null && qbp.getClauseNamesForDest().size() == 2) {
          // For multi-insert query, currently we only optimize the FROM clause.
          // Hence, introduce multi-insert token on top of it.
          // However, first we need to reset existing token (insert).
          // Using qbp.getClauseNamesForDest().size() >= 2 would be
          // equivalent, but we use == to avoid setting the property
          // multiple times
          plannerCtx.resetToken();
          plannerCtx.setMultiInsertToken((ASTNode) qbp.getQueryFrom().getChild(0));
        }
        break;

      case HiveParser.TOK_FROM:
        int child_count = ast.getChildCount();
        if (child_count != 1) {
          throw new SemanticException(generateErrorMessage(ast,
              "Multiple Children " + child_count));
        }

        if (!qbp.getIsSubQ()) {
          qbp.setQueryFromExpr(ast);
        }

        // Check if this is a subquery / lateral view
        ASTNode frm = (ASTNode) ast.getChild(0);
        if (frm.getToken().getType() == HiveParser.TOK_TABREF) {
          processTable(qb, frm);
        } else if (frm.getToken().getType() == HiveParser.TOK_SUBQUERY) {
          processSubQuery(qb, frm);
        } else if (frm.getToken().getType() == HiveParser.TOK_LATERAL_VIEW ||
            frm.getToken().getType() == HiveParser.TOK_LATERAL_VIEW_OUTER) {
          queryProperties.setHasLateralViews(true);
          processLateralView(qb, frm);
        } else if (isJoinToken(frm)) {
          processJoin(qb, frm);
          qbp.setJoinExpr(frm);
        }else if(frm.getToken().getType() == HiveParser.TOK_PTBLFUNCTION){
          queryProperties.setHasPTF(true);
          processPTF(qb, frm);
        }
        break;

      case HiveParser.TOK_CLUSTERBY:
        // Get the clusterby aliases - these are aliased to the entries in the
        // select list
        queryProperties.setHasClusterBy(true);
        qbp.setClusterByExprForClause(ctx_1.dest, ast);
        break;

      case HiveParser.TOK_DISTRIBUTEBY:
        // Get the distribute by aliases - these are aliased to the entries in
        // the
        // select list
        queryProperties.setHasDistributeBy(true);
        qbp.setDistributeByExprForClause(ctx_1.dest, ast);
        if (qbp.getClusterByForClause(ctx_1.dest) != null) {
          throw new SemanticException(generateErrorMessage(ast,
              ErrorMsg.CLUSTERBY_DISTRIBUTEBY_CONFLICT.getMsg()));
        } else if (qbp.getOrderByForClause(ctx_1.dest) != null) {
          throw new SemanticException(generateErrorMessage(ast,
              ErrorMsg.ORDERBY_DISTRIBUTEBY_CONFLICT.getMsg()));
        }
        break;

      case HiveParser.TOK_SORTBY:
        // Get the sort by aliases - these are aliased to the entries in the
        // select list
        queryProperties.setHasSortBy(true);
        qbp.setSortByExprForClause(ctx_1.dest, ast);
        if (qbp.getClusterByForClause(ctx_1.dest) != null) {
          throw new SemanticException(generateErrorMessage(ast,
              ErrorMsg.CLUSTERBY_SORTBY_CONFLICT.getMsg()));
        } else if (qbp.getOrderByForClause(ctx_1.dest) != null) {
          throw new SemanticException(generateErrorMessage(ast,
              ErrorMsg.ORDERBY_SORTBY_CONFLICT.getMsg()));
        }

        break;

      case HiveParser.TOK_ORDERBY:
        // Get the order by aliases - these are aliased to the entries in the
        // select list
        queryProperties.setHasOrderBy(true);
        qbp.setOrderByExprForClause(ctx_1.dest, ast);
        if (qbp.getClusterByForClause(ctx_1.dest) != null) {
          throw new SemanticException(generateErrorMessage(ast,
              ErrorMsg.CLUSTERBY_ORDERBY_CONFLICT.getMsg()));
        }
        // If there are aggregations in order by, we need to remember them in qb.
        qbp.addAggregationExprsForClause(ctx_1.dest,
            doPhase1GetAggregationsFromSelect(ast, qb, ctx_1.dest));
        break;

      case HiveParser.TOK_GROUPBY:
      case HiveParser.TOK_ROLLUP_GROUPBY:
      case HiveParser.TOK_CUBE_GROUPBY:
      case HiveParser.TOK_GROUPING_SETS:
        // Get the groupby aliases - these are aliased to the entries in the
        // select list
        queryProperties.setHasGroupBy(true);
        if (qbp.getJoinExpr() != null) {
          queryProperties.setHasJoinFollowedByGroupBy(true);
        }
        if (qbp.getSelForClause(ctx_1.dest).getToken().getType() == HiveParser.TOK_SELECTDI) {
          throw new SemanticException(generateErrorMessage(ast,
              ErrorMsg.SELECT_DISTINCT_WITH_GROUPBY.getMsg()));
        }
        qbp.setGroupByExprForClause(ctx_1.dest, ast);
        skipRecursion = true;

        // Rollup and Cubes are syntactic sugar on top of grouping sets
        if (ast.getToken().getType() == HiveParser.TOK_ROLLUP_GROUPBY) {
          qbp.getDestRollups().add(ctx_1.dest);
        } else if (ast.getToken().getType() == HiveParser.TOK_CUBE_GROUPBY) {
          qbp.getDestCubes().add(ctx_1.dest);
        } else if (ast.getToken().getType() == HiveParser.TOK_GROUPING_SETS) {
          qbp.getDestGroupingSets().add(ctx_1.dest);
        }
        break;

      case HiveParser.TOK_HAVING:
        qbp.setHavingExprForClause(ctx_1.dest, ast);
        qbp.addAggregationExprsForClause(ctx_1.dest,
            doPhase1GetAggregationsFromSelect(ast, qb, ctx_1.dest));
        break;

      case HiveParser.KW_WINDOW:
        if (!qb.hasWindowingSpec(ctx_1.dest) ) {
          throw new SemanticException(generateErrorMessage(ast,
              "Query has no Cluster/Distribute By; but has a Window definition"));
        }
        handleQueryWindowClauses(qb, ctx_1, ast);
        break;

      case HiveParser.TOK_LIMIT:
        if (ast.getChildCount() == 2) {
          qbp.setDestLimit(ctx_1.dest,
              new Integer(ast.getChild(0).getText()),
              new Integer(ast.getChild(1).getText()));
        } else {
          qbp.setDestLimit(ctx_1.dest, new Integer(0),
              new Integer(ast.getChild(0).getText()));
        }
        break;

      case HiveParser.TOK_ANALYZE:
        // Case of analyze command

        String table_name = getUnescapedName((ASTNode) ast.getChild(0).getChild(0)).toLowerCase();


        qb.setTabAlias(table_name, table_name);
        qb.addAlias(table_name);
        qb.getParseInfo().setIsAnalyzeCommand(true);
        qb.getParseInfo().setNoScanAnalyzeCommand(this.noscan);
        // Allow analyze the whole table and dynamic partitions
        HiveConf.setVar(conf, HiveConf.ConfVars.DYNAMICPARTITIONINGMODE, "nonstrict");
        HiveConf.setVar(conf, HiveConf.ConfVars.HIVEMAPREDMODE, "nonstrict");

        break;

      case HiveParser.TOK_UNIONALL:
        if (!qbp.getIsSubQ()) {
          // this shouldn't happen. The parser should have converted the union to be
          // contained in a subquery. Just in case, we keep the error as a fallback.
          throw new SemanticException(generateErrorMessage(ast,
              ErrorMsg.UNION_NOTIN_SUBQ.getMsg()));
        }
        skipRecursion = false;
        break;

      case HiveParser.TOK_INSERT:
        ASTNode destination = (ASTNode) ast.getChild(0);
        Tree tab = destination.getChild(0);

        // Proceed if AST contains partition & If Not Exists
        if (destination.getChildCount() == 2 &&
            tab.getChildCount() == 2 &&
            destination.getChild(1).getType() == HiveParser.TOK_IFNOTEXISTS) {
          String tableName = tab.getChild(0).getChild(0).getText();

          Tree partitions = tab.getChild(1);
          int childCount = partitions.getChildCount();
          HashMap<String, String> partition = new HashMap<String, String>();
          for (int i = 0; i < childCount; i++) {
            String partitionName = partitions.getChild(i).getChild(0).getText();
            // Convert to lowercase for the comparison
            partitionName = partitionName.toLowerCase();
            Tree pvalue = partitions.getChild(i).getChild(1);
            if (pvalue == null) {
              break;
            }
            String partitionVal = stripQuotes(pvalue.getText());
            partition.put(partitionName, partitionVal);
          }
          // if it is a dynamic partition throw the exception
          if (childCount != partition.size()) {
            throw new SemanticException(ErrorMsg.INSERT_INTO_DYNAMICPARTITION_IFNOTEXISTS
                .getMsg(partition.toString()));
          }
          Table table = null;
          try {
            table = this.getTableObjectByName(tableName);
          } catch (HiveException ex) {
            throw new SemanticException(ex);
          }
          try {
            Partition parMetaData = db.getPartition(table, partition, false);
            // Check partition exists if it exists skip the overwrite
            if (parMetaData != null) {
              phase1Result = false;
              skipRecursion = true;
              LOG.info("Partition already exists so insert into overwrite " +
                  "skipped for partition : " + parMetaData.toString());
              break;
            }
          } catch (HiveException e) {
            LOG.info("Error while getting metadata : ", e);
          }
          validatePartSpec(table, partition, (ASTNode)tab, conf, false);
        }
        skipRecursion = false;
        break;
      case HiveParser.TOK_LATERAL_VIEW:
      case HiveParser.TOK_LATERAL_VIEW_OUTER:
        // todo: nested LV
        assert ast.getChildCount() == 1;
        qb.getParseInfo().getDestToLateralView().put(ctx_1.dest, ast);
        break;
      case HiveParser.TOK_CTE:
        processCTE(qb, ast);
        break;
      default:
        skipRecursion = false;
        break;
      }
    }

    if (!skipRecursion) {
      // Iterate over the rest of the children
      // 对于一个TOK中会出现多个统计子TOK的情况，如两个TOK_SUBQUERY,或者TOK_FROM和TOK_INSERT
      int child_count = ast.getChildCount();
      for (int child_pos = 0; child_pos < child_count && phase1Result; ++child_pos) {
        // Recurse
        // 当当前TOK解析正确时，则继续解析子TOK
        phase1Result = phase1Result && doPhase1(
            (ASTNode)ast.getChild(child_pos), qb, ctx_1, plannerCtx);
      }
    }
    return phase1Result;
  }
```
