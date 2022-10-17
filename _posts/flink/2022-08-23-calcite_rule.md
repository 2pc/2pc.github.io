---
title: FlinkSQL(Flink1.15)规则优化以及Calcite原理
tagline: ""
category : Flink1.15
layout: post
tags : [flink, Calcite, realtime]
---
SQL语句示例
```
select p.id,o.id from products p join orders o on p.id=o.id where p.id > 5
```
优化前，从SqlNode到RelNode阶段，从SqlToRelConverter.convertQuery的trace日志
```
[DEBUG] 2022-08-22 14:50:41,662(627) --> [main] org.apache.calcite.sql2rel.SqlToRelConverter.convertQuery(SqlToRelConverter.java:576): Plan after converting SqlNode to RelNode
LogicalProject(ID=[$0], ID0=[$3])
  LogicalFilter(condition=[>($0, 1)])
    LogicalJoin(condition=[=($0, $3)], joinType=[inner])
      LogicalTableScan(table=[[PRODUCTS]])
      LogicalTableScan(table=[[ORDERS]])
```
优化后
```
LogicalProject(ID=[$0], ID0=[$3])
  LogicalJoin(condition=[=($0, $3)], joinType=[inner])
    LogicalFilter(condition=[>($0, 5)])
      EnumerableTableScan(table=[[PRODUCTS]])
    EnumerableTableScan(table=[[ORDERS]])
```
SQL示例2
```
select u.id as user_id, u.name as user_name, w.content as content, u.age as user_age from users u"
            + " join weibos  w on u.id=w.id where u.age > 30 and w.id>10 order by user_id
```
优化前,还是从SqlNode到RelNode阶段
```
[DEBUG] 2022-08-22 15:13:44,542(552) --> [main] org.apache.calcite.sql2rel.SqlToRelConverter.convertQuery(SqlToRelConverter.java:576): Plan after converting SqlNode to RelNode
LogicalSort(sort0=[$0], dir0=[ASC])
  LogicalProject(USER_ID=[$0], USER_NAME=[$1], CONTENT=[$5], USER_AGE=[$2])
    LogicalFilter(condition=[AND(>($2, 30), >($3, 10))])
      LogicalJoin(condition=[=($0, $3)], joinType=[inner])
        LogicalTableScan(table=[[USERS]])
        LogicalTableScan(table=[[WEIBOS]])
```
优化后，因为两个表都有条件，生成了两个LogicalFilter
```
LogicalSort(sort0=[$0], dir0=[ASC])
  LogicalProject(USER_ID=[$0], USER_NAME=[$1], CONTENT=[$5], USER_AGE=[$2])
    LogicalJoin(condition=[=($0, $3)], joinType=[inner])
      LogicalFilter(condition=[>($2, 30)])//
        EnumerableTableScan(table=[[USERS]])
      LogicalFilter(condition=[>($0, 10)])
        EnumerableTableScan(table=[[WEIBOS]])
//

```
trace日志
```
[TRACE] 2022-08-22 14:50:41,737(702) --> [main] org.apache.calcite.plan.hep.HepPlanner.dumpGraph(HepPlanner.java:1015): 
//filter没下推
Breadth-first from root:  {
    HepRelVertex#18 = rel#17:LogicalProject(input=HepRelVertex#16,ID=$0,ID0=$3), rowcount=750.0, cumulative cost=3200.0
    HepRelVertex#16 = rel#15:LogicalFilter(input=HepRelVertex#14,condition=>($0, 1)), rowcount=750.0, cumulative cost=2450.0
    HepRelVertex#14 = rel#13:LogicalJoin(left=HepRelVertex#11,right=HepRelVertex#12,condition==($0, $3),joinType=inner), rowcount=1500.0, cumulative cost=1700.0
    HepRelVertex#11 = rel#6:EnumerableTableScan(table=[PRODUCTS]), rowcount=100.0, cumulative cost=100.0
    HepRelVertex#12 = rel#7:EnumerableTableScan(table=[ORDERS]), rowcount=100.0, cumulative cost=100.0
}  
[TRACE] 2022-08-22 14:50:41,737(702) --> [main] org.apache.calcite.plan.hep.HepPlanner.applyRules(HepPlanner.java:402): Applying rule set [FilterJoinRule:FilterJoinRule:filter]  
[TRACE] 2022-08-22 14:50:41,737(702) --> [main] org.apache.calcite.plan.hep.HepPlanner.collectGarbage(HepPlanner.java:944): collecting garbage  
//FilterJoinRule:FilterJoinRule规则
[DEBUG] 2022-08-22 14:50:41,740(705) --> [main] org.apache.calcite.plan.AbstractRelOptPlanner.fireRule(AbstractRelOptPlanner.java:305): call#0: Apply rule [FilterJoinRule:FilterJoinRule:filter] to [rel#15:LogicalFilter(input=HepRelVertex#14,condition=>($0, 1)), rel#13:LogicalJoin(left=HepRelVertex#11,right=HepRelVertex#12,condition==($0, $3),joinType=inner)]  
[TRACE] 2022-08-22 14:54:56,202(255167) --> [main] org.apache.calcite.rel.AbstractRelNode.<init>(AbstractRelNode.java:116): new LogicalFilter#19  
[TRACE] 2022-08-22 14:54:56,208(255173) --> [main] org.apache.calcite.rel.AbstractRelNode.<init>(AbstractRelNode.java:116): new LogicalJoin#20  
//FilterJoinRule:FilterJoinRule规则
[DEBUG] 2022-08-22 14:54:56,209(255174) --> [main] org.apache.calcite.plan.AbstractRelOptPlanner.notifyTransformation(AbstractRelOptPlanner.java:345): call#0: Rule FilterJoinRule:FilterJoinRule:filter arguments [rel#15:LogicalFilter(input=HepRelVertex#14,condition=>($0, 1)), rel#13:LogicalJoin(left=HepRelVertex#11,right=HepRelVertex#12,condition==($0, $3),joinType=inner)] produced LogicalJoin#20  
[TRACE] 2022-08-22 14:54:56,228(255193) --> [main] org.apache.calcite.rel.AbstractRelNode.<init>(AbstractRelNode.java:116): new HepRelVertex#21  
[TRACE] 2022-08-22 14:54:56,228(255193) --> [main] org.apache.calcite.rel.AbstractRelNode.<init>(AbstractRelNode.java:116): new LogicalJoin#22  
[TRACE] 2022-08-22 14:54:56,228(255193) --> [main] org.apache.calcite.rel.AbstractRelNode.<init>(AbstractRelNode.java:116): new HepRelVertex#23  
[TRACE] 2022-08-22 14:54:56,230(255195) --> [main] org.apache.calcite.plan.hep.HepPlanner.dumpGraph(HepPlanner.java:1015): 
//filter下推后
Breadth-first from root:  {
    HepRelVertex#18 = rel#17:LogicalProject(input=HepRelVertex#16,ID=$0,ID0=$3), rowcount=750.0, cumulative cost=1750.0
    HepRelVertex#23 = rel#22:LogicalJoin(left=HepRelVertex#21,right=HepRelVertex#12,condition==($0, $3),joinType=inner), rowcount=750.0, cumulative cost=1000.0
    HepRelVertex#21 = rel#19:LogicalFilter(input=HepRelVertex#11,condition=>($0, 1)), rowcount=50.0, cumulative cost=150.0
    HepRelVertex#12 = rel#7:EnumerableTableScan(table=[ORDERS]), rowcount=100.0, cumulative cost=100.0
    HepRelVertex#11 = rel#6:EnumerableTableScan(table=[PRODUCTS]), rowcount=100.0, cumulative cost=100.0
}  
```
看下原理,首先是优化器，使用HepPlanner优化器，并添加规则FilterIntoJoinRule
```
HepProgramBuilder builder = new HepProgramBuilder();
//这里添加一条规则
builder.addRuleInstance(FilterJoinRule.FilterIntoJoinRule.FILTER_ON_JOIN);
HepPlanner planner = new HepPlanner(builder.build());
```
优化调用 planner的findBestExp()方法
```
  // implement RelOptPlanner
  public RelNode findBestExp() {
    assert root != null;

    executeProgram(mainProgram);

    // Get rid of everything except what's in the final plan.
    collectGarbage();

    return buildFinalPlan(root);
  }

  private void executeProgram(HepProgram program) {
    HepProgram savedProgram = currentProgram;
    currentProgram = program;
    currentProgram.initialize(program == mainProgram);
    for (HepInstruction instruction : currentProgram.instructions) {
      instruction.execute(this);//RuleInstance这里比较关键
      int delta = nTransformations - nTransformationsLastGC;
      if (delta > graphSizeLastGC) {
        // The number of transformations performed since the last
        // garbage collection is greater than the number of vertices in
        // the graph at that time.  That means there should be a
        // reasonable amount of garbage to collect now.  We do it this
        // way to amortize garbage collection cost over multiple
        // instructions, while keeping the highwater memory usage
        // proportional to the graph size.
        collectGarbage();
      }
    }
    currentProgram = savedProgram;
  }
```
RuleInstance.execute
```
static class RuleInstance extends HepInstruction {
/**
 * Description to look for, or null if rule specified explicitly.
 */
String ruleDescription;

/**
 * Explicitly specified rule, or rule looked up by planner from
 * description.
 */
RelOptRule rule;

void initialize(boolean clearCache) {
  if (!clearCache) {
    return;
  }

  if (ruleDescription != null) {
    // Look up anew each run.
    rule = null;
  }
}
//这里自然是HepPlanner
void execute(HepPlanner planner) {
  planner.executeInstruction(this);
}
}
```
HepPlanner.executeInstruction<br />applyRules，看名字就是与配置的规则有关了
```
void executeInstruction(
  HepInstruction.RuleInstance instruction) {
if (skippingGroup()) {
  return;
}
if (instruction.rule == null) {
  assert instruction.ruleDescription != null;
  instruction.rule =
      getRuleByDescription(instruction.ruleDescription);
  LOGGER.trace("Looking up rule with description {}, found {}",
      instruction.ruleDescription, instruction.rule);
}
//applyRules，看名字就是与配置的规则有关了
if (instruction.rule != null) {
  applyRules(
      Collections.singleton(instruction.rule),
      true);
}
}
```
HepPlanner.applyRules
```
private void applyRules(
  Collection<RelOptRule> rules,
  boolean forceConversions) {
if (currentProgram.group != null) {
  assert currentProgram.group.collecting;
  currentProgram.group.ruleSet.addAll(rules);
  return;
}

LOGGER.trace("Applying rule set {}", rules);

boolean fullRestartAfterTransformation =
    currentProgram.matchOrder != HepMatchOrder.ARBITRARY
    && currentProgram.matchOrder != HepMatchOrder.DEPTH_FIRST;

int nMatches = 0;

boolean fixedPoint;
do {
  Iterator<HepRelVertex> iter = getGraphIterator(root);
  fixedPoint = true;
  while (iter.hasNext()) {
    HepRelVertex vertex = iter.next();
    for (RelOptRule rule : rules) {
      //
      HepRelVertex newVertex =
          applyRule(rule, vertex, forceConversions);
      if (newVertex == null || newVertex == vertex) {
        continue;
      }
      ++nMatches;
      if (nMatches >= currentProgram.matchLimit) {
        return;
      }
      if (fullRestartAfterTransformation) {
        iter = getGraphIterator(root);
      } else {
        // To the extent possible, pick up where we left
        // off; have to create a new iterator because old
        // one was invalidated by transformation.
        iter = getGraphIterator(newVertex);
        if (currentProgram.matchOrder == HepMatchOrder.DEPTH_FIRST) {
          nMatches =
              depthFirstApply(iter, rules, forceConversions, nMatches);
          if (nMatches >= currentProgram.matchLimit) {
            return;
          }
        }
        // Remember to go around again since we're
        // skipping some stuff.
        fixedPoint = false;
      }
      break;
    }
  }
} while (!fixedPoint);
}
```
HepPlanner.applyRule<br />这里的规则是FilterIntoJoinRule，既不是ConverterRule也不是CommonRelSubExprRule类型
```
  private HepRelVertex applyRule(
      RelOptRule rule,
      HepRelVertex vertex,
      boolean forceConversions) {
    if (!belongsToDag(vertex)) {
      return null;
    }
    RelTrait parentTrait = null;
    List<RelNode> parents = null;
    if (rule instanceof ConverterRule) {
      // Guaranteed converter rules require special casing to make sure
      // they only fire where actually needed, otherwise they tend to
      // fire to infinity and beyond.
      ConverterRule converterRule = (ConverterRule) rule;
      if (converterRule.isGuaranteed() || !forceConversions) {
        if (!doesConverterApply(converterRule, vertex)) {
          return null;
        }
        parentTrait = converterRule.getOutTrait();
      }
    } else if (rule instanceof CommonRelSubExprRule) {
      // Only fire CommonRelSubExprRules if the vertex is a common
      // subexpression.
      List<HepRelVertex> parentVertices = getVertexParents(vertex);
      if (parentVertices.size() < 2) {
        return null;
      }
      parents = new ArrayList<>();
      for (HepRelVertex pVertex : parentVertices) {
        parents.add(pVertex.getCurrentRel());
      }
    }

    final List<RelNode> bindings = new ArrayList<>();
    final Map<RelNode, List<RelNode>> nodeChildren = new HashMap<>();
    boolean match =
        matchOperands(
            rule.getOperand(),
            vertex.getCurrentRel(),
            bindings,
            nodeChildren);

    if (!match) {
      return null;
    }

    HepRuleCall call =
        new HepRuleCall(
            this,
            rule.getOperand(),
            bindings.toArray(new RelNode[0]),
            nodeChildren,
            parents);

    // Allow the rule to apply its own side-conditions.
    if (!rule.matches(call)) {
      return null;
    }
	//规则在这里匹配出发
    fireRule(call);

    if (!call.getResults().isEmpty()) {
      return applyTransformationResults(
          vertex,
          call,
          parentTrait);
    }

    return null;
  }
```
AbstractRelOptPlanner.fireRule
```
  protected void fireRule(
      RelOptRuleCall ruleCall) {
    checkCancel();

    assert ruleCall.getRule().matches(ruleCall);
    if (isRuleExcluded(ruleCall.getRule())) {
      LOGGER.debug("call#{}: Rule [{}] not fired due to exclusion filter",
          ruleCall.id, ruleCall.getRule());
      return;
    }

    if (LOGGER.isDebugEnabled()) {
      // Leave this wrapped in a conditional to prevent unnecessarily calling Arrays.toString(...)
      LOGGER.debug("call#{}: Apply rule [{}] to {}",
          ruleCall.id, ruleCall.getRule(), Arrays.toString(ruleCall.rels));
    }

    if (listener != null) {
      RelOptListener.RuleAttemptedEvent event =
          new RelOptListener.RuleAttemptedEvent(
              this,
              ruleCall.rel(0),
              ruleCall,
              true);
      listener.ruleAttempted(event);
    }
	//匹配到规则FilterIntoJoinRule
    ruleCall.getRule().onMatch(ruleCall);

    if (listener != null) {
      RelOptListener.RuleAttemptedEvent event =
          new RelOptListener.RuleAttemptedEvent(
              this,
              ruleCall.rel(0),
              ruleCall,
              false);
      listener.ruleAttempted(event);
    }
  }
```
这里配置到之前配置的规则：FilterIntoJoinRule<br />分别取出filter和join后执行perform
```
  public static class FilterIntoJoinRule extends FilterJoinRule {
    public FilterIntoJoinRule(boolean smart,
        RelBuilderFactory relBuilderFactory, Predicate predicate) {
      super(
          operand(Filter.class,
              operand(Join.class, RelOptRule.any())),
          "FilterJoinRule:filter", smart, relBuilderFactory,
          predicate);
    }

    @Deprecated // to be removed before 2.0
    public FilterIntoJoinRule(boolean smart,
        RelFactories.FilterFactory filterFactory,
        RelFactories.ProjectFactory projectFactory,
        Predicate predicate) {
      this(smart, RelBuilder.proto(filterFactory, projectFactory), predicate);
    }
	//分别取出filter和join
    @Override public void onMatch(RelOptRuleCall call) {
      Filter filter = call.rel(0);
      Join join = call.rel(1);
      perform(call, filter, join);
    }
  }
```
perform由父类FilterJoinRule实现<br />代码比较长，主要是从左右两边表解析出filter<br />在生成RelNode：LogicalFilter,在重新生成joinRelNode：LogicalJoin
```
  protected void perform(RelOptRuleCall call, Filter filter,
      Join join) {
    final List<RexNode> joinFilters =
        RelOptUtil.conjunctions(join.getCondition());
    final List<RexNode> origJoinFilters = ImmutableList.copyOf(joinFilters);

    // If there is only the joinRel,
    // make sure it does not match a cartesian product joinRel
    // (with "true" condition), otherwise this rule will be applied
    // again on the new cartesian product joinRel.
    if (filter == null && joinFilters.isEmpty()) {
      return;
    }

    final List<RexNode> aboveFilters =
        filter != null
            ? conjunctions(filter.getCondition())
            : new ArrayList<>();
    final ImmutableList<RexNode> origAboveFilters =
        ImmutableList.copyOf(aboveFilters);

    // Simplify Outer Joins
    JoinRelType joinType = join.getJoinType();
    if (smart
        && !origAboveFilters.isEmpty()
        && join.getJoinType() != JoinRelType.INNER) {
      joinType = RelOptUtil.simplifyJoin(join, origAboveFilters, joinType);
    }

    final List<RexNode> leftFilters = new ArrayList<>();
    final List<RexNode> rightFilters = new ArrayList<>();

    // TODO - add logic to derive additional filters.  E.g., from
    // (t1.a = 1 AND t2.a = 2) OR (t1.b = 3 AND t2.b = 4), you can
    // derive table filters:
    // (t1.a = 1 OR t1.b = 3)
    // (t2.a = 2 OR t2.b = 4)

    // Try to push down above filters. These are typically where clause
    // filters. They can be pushed down if they are not on the NULL
    // generating side.
    boolean filterPushed = false;
    if (RelOptUtil.classifyFilters(
        join,
        aboveFilters,
        joinType,
        !(join instanceof EquiJoin),
        !joinType.generatesNullsOnLeft(),
        !joinType.generatesNullsOnRight(),
        joinFilters,
        leftFilters,
        rightFilters)) {
      filterPushed = true;
    }

    // Move join filters up if needed
    validateJoinFilters(aboveFilters, joinFilters, join, joinType);

    // If no filter got pushed after validate, reset filterPushed flag
    if (leftFilters.isEmpty()
        && rightFilters.isEmpty()
        && joinFilters.size() == origJoinFilters.size()) {
      if (Sets.newHashSet(joinFilters)
          .equals(Sets.newHashSet(origJoinFilters))) {
        filterPushed = false;
      }
    }

    // Try to push down filters in ON clause. A ON clause filter can only be
    // pushed down if it does not affect the non-matching set, i.e. it is
    // not on the side which is preserved.
    if (RelOptUtil.classifyFilters(
        join,
        joinFilters,
        joinType,
        false,
        !joinType.generatesNullsOnRight(),
        !joinType.generatesNullsOnLeft(),
        joinFilters,
        leftFilters,
        rightFilters)) {
      filterPushed = true;
    }

    // if nothing actually got pushed and there is nothing leftover,
    // then this rule is a no-op
    if ((!filterPushed
            && joinType == join.getJoinType())
        || (joinFilters.isEmpty()
            && leftFilters.isEmpty()
            && rightFilters.isEmpty())) {
      return;
    }

    // create Filters on top of the children if any filters were
    // pushed to them
    //生成左右表LogicalFilter节点
    final RexBuilder rexBuilder = join.getCluster().getRexBuilder();
    final RelBuilder relBuilder = call.builder();
    final RelNode leftRel =
        relBuilder.push(join.getLeft()).filter(leftFilters).build();
    final RelNode rightRel =
        relBuilder.push(join.getRight()).filter(rightFilters).build();

    // create the new join node referencing the new children and
    // containing its new join filters (if there are any)
    final ImmutableList<RelDataType> fieldTypes =
        ImmutableList.<RelDataType>builder()
            .addAll(RelOptUtil.getFieldTypeList(leftRel.getRowType()))
            .addAll(RelOptUtil.getFieldTypeList(rightRel.getRowType())).build();
    final RexNode joinFilter =
        RexUtil.composeConjunction(rexBuilder,
            RexUtil.fixUp(rexBuilder, joinFilters, fieldTypes));

    // If nothing actually got pushed and there is nothing leftover,
    // then this rule is a no-op
    if (joinFilter.isAlwaysTrue()
        && leftFilters.isEmpty()
        && rightFilters.isEmpty()
        && joinType == join.getJoinType()) {
      return;
    }
	//重新生成LogicalJoin
    RelNode newJoinRel =
        join.copy(
            join.getTraitSet(),
            joinFilter,
            leftRel,
            rightRel,
            joinType,
            join.isSemiJoinDone());
    call.getPlanner().onCopy(join, newJoinRel);
    if (!leftFilters.isEmpty()) {
      call.getPlanner().onCopy(filter, leftRel);
    }
    if (!rightFilters.isEmpty()) {
      call.getPlanner().onCopy(filter, rightRel);
    }

    relBuilder.push(newJoinRel);

    // Create a project on top of the join if some of the columns have become
    // NOT NULL due to the join-type getting stricter.
    //生成LogicalProject
    relBuilder.convert(join.getRowType(), false);

    // create a FilterRel on top of the join if needed
    relBuilder.filter(
        RexUtil.fixUp(rexBuilder, aboveFilters,
            RelOptUtil.getFieldTypeList(relBuilder.peek().getRowType())));

    call.transformTo(relBuilder.build());
  }
```
HepPlanner.applyTransformationResults<br />用新的LogicalJoin替换掉原LogicFilter，添加到 HepPlanner 的 graph  新DAG
```
  private HepRelVertex applyTransformationResults(
      HepRelVertex vertex,
      HepRuleCall call,
      RelTrait parentTrait) {
    // TODO jvs 5-Apr-2006:  Take the one that gives the best
    // global cost rather than the best local cost.  That requires
    // "tentative" graph edits.

    assert !call.getResults().isEmpty();

    RelNode bestRel = null;

    if (call.getResults().size() == 1) {
      // No costing required; skip it to minimize the chance of hitting
      // rels without cost information.
      bestRel = call.getResults().get(0);
    } else {
      RelOptCost bestCost = null;
      final RelMetadataQuery mq = call.getMetadataQuery();
      for (RelNode rel : call.getResults()) {
        RelOptCost thisCost = getCost(rel, mq);
        if (LOGGER.isTraceEnabled()) {
          // Keep in the isTraceEnabled for the getRowCount method call
          LOGGER.trace("considering {} with cumulative cost={} and rowcount={}",
              rel, thisCost, mq.getRowCount(rel));
        }
        if ((bestRel == null) || thisCost.isLt(bestCost)) {
          bestRel = rel;
          bestCost = thisCost;
        }
      }
    }

    ++nTransformations;
    notifyTransformation(
        call,
        bestRel,
        true);

    // Before we add the result, make a copy of the list of vertex's
    // parents.  We'll need this later during contraction so that
    // we only update the existing parents, not the new parents
    // (otherwise loops can result).  Also take care of filtering
    // out parents by traits in case we're dealing with a converter rule.
    final List<HepRelVertex> allParents =
        Graphs.predecessorListOf(graph, vertex);
    final List<HepRelVertex> parents = new ArrayList<>();
    for (HepRelVertex parent : allParents) {
      if (parentTrait != null) {
        RelNode parentRel = parent.getCurrentRel();
        if (parentRel instanceof Converter) {
          // We don't support automatically chaining conversions.
          // Treating a converter as a candidate parent here
          // can cause the "iParentMatch" check below to
          // throw away a new converter needed in
          // the multi-parent DAG case.
          continue;
        }
        if (!parentRel.getTraitSet().contains(parentTrait)) {
          // This parent does not want the converted result.
          continue;
        }
      }
      parents.add(parent);
    }

    HepRelVertex newVertex = addRelToGraph(bestRel);

    // There's a chance that newVertex is the same as one
    // of the parents due to common subexpression recognition
    // (e.g. the LogicalProject added by JoinCommuteRule).  In that
    // case, treat the transformation as a nop to avoid
    // creating a loop.
    int iParentMatch = parents.indexOf(newVertex);
    if (iParentMatch != -1) {
      newVertex = parents.get(iParentMatch);
    } else {
      contractVertices(newVertex, vertex, parents);
    }

    if (getListener() != null) {
      // Assume listener doesn't want to see garbage.
      collectGarbage();
    }

    notifyTransformation(
        call,
        bestRel,
        false);

    dumpGraph();

    return newVertex;
  }
```

规则<br />FlinkStreamProgram<br />FlinkVolcanoProgram主要两个FlinkStreamRuleSets.LOGICAL_OPT_RULES,<br />FlinkStreamRuleSets.PHYSICAL_OPT_RULE<br />别看只有两个，这都是代表一系列的规则
```
// optimize the logical plan
chainedProgram.addLast(
  LOGICAL,
  FlinkVolcanoProgramBuilder.newBuilder
    .add(FlinkStreamRuleSets.LOGICAL_OPT_RULES)
    .setRequiredOutputTraits(Array(FlinkConventions.LOGICAL))
    .build()
)
// optimize the physical plan
chainedProgram.addLast(
  PHYSICAL,
  FlinkVolcanoProgramBuilder.newBuilder
    .add(FlinkStreamRuleSets.PHYSICAL_OPT_RULES)
    .setRequiredOutputTraits(Array(FlinkConventions.STREAM_PHYSICAL))
    .build()
)
```
逻辑执行计划<br />LOGICAL_CONVERTERS负责转换RelNode到FlinkLogicalRel转换
```
/** RuleSet to do logical optimize for stream */
val LOGICAL_OPT_RULES: RuleSet = RuleSets.ofList(
(
  FILTER_RULES.asScala ++
    PROJECT_RULES.asScala ++
    PRUNE_EMPTY_RULES.asScala ++
    LOGICAL_RULES.asScala ++
    LOGICAL_CONVERTERS.asScala//转换RelNode -->FlinkLogicalRel
).asJava)
```
LOGICAL_CONVERTERS
```
/** RuleSet to translate calcite nodes to flink nodes */
private val LOGICAL_CONVERTERS: RuleSet = RuleSets.ofList(
// translate to flink logical rel nodes
FlinkLogicalAggregate.STREAM_CONVERTER,
FlinkLogicalTableAggregate.CONVERTER,
FlinkLogicalOverAggregate.CONVERTER,
FlinkLogicalCalc.CONVERTER,
FlinkLogicalCorrelate.CONVERTER,
FlinkLogicalJoin.CONVERTER,
FlinkLogicalSort.STREAM_CONVERTER,
FlinkLogicalUnion.CONVERTER,
FlinkLogicalValues.CONVERTER,
FlinkLogicalTableSourceScan.CONVERTER,
FlinkLogicalLegacyTableSourceScan.CONVERTER,
FlinkLogicalTableFunctionScan.CONVERTER,
FlinkLogicalDataStreamTableScan.CONVERTER,
FlinkLogicalIntermediateTableScan.CONVERTER,
FlinkLogicalExpand.CONVERTER,
FlinkLogicalRank.CONVERTER,
FlinkLogicalWatermarkAssigner.CONVERTER,
FlinkLogicalWindowAggregate.CONVERTER,
FlinkLogicalWindowTableAggregate.CONVERTER,
FlinkLogicalSnapshot.CONVERTER,
FlinkLogicalMatch.CONVERTER,
FlinkLogicalSink.CONVERTER,
FlinkLogicalLegacySink.CONVERTER
)
```
比如FlinkLogicalJoinConverter负责join的转换<br />即join RelNode-->FlinkLogicalJoin
```
/** Support all joins. */
private class FlinkLogicalJoinConverter
  extends ConverterRule(
    classOf[LogicalJoin],
    Convention.NONE,
    FlinkConventions.LOGICAL,
    "FlinkLogicalJoinConverter") {

  override def convert(rel: RelNode): RelNode = {
    val join = rel.asInstanceOf[LogicalJoin]
    val newLeft = RelOptRule.convert(join.getLeft, FlinkConventions.LOGICAL)
    val newRight = RelOptRule.convert(join.getRight, FlinkConventions.LOGICAL)
    FlinkLogicalJoin.create(newLeft, newRight, join.getCondition, join.getHints, join.getJoinType)
  }
}

object FlinkLogicalJoin {
  val CONVERTER: ConverterRule = new FlinkLogicalJoinConverter
//创建FlinkLogicalJoin
  def create(
      left: RelNode,
      right: RelNode,
      conditionExpr: RexNode,
      hints: JList[RelHint],
      joinType: JoinRelType): FlinkLogicalJoin = {
    val cluster = left.getCluster
    val traitSet = cluster.traitSetOf(FlinkConventions.LOGICAL).simplify()
    new FlinkLogicalJoin(cluster, traitSet, left, right, conditionExpr, hints, joinType)
  }
}
```
