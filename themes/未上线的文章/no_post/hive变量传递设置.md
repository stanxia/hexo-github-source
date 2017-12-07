---
title: hive变量传递设置
date: 2017-04-26 21:51:57
tags: hive
categories: 大数据
---

在oozie的workflow中执行一个hive查询，但是直接就报异常：Variable substitution depth too large:40，从网上查询可知，可以确认是由于语句中使用了过多的变量导致，在hive以前的版本中，这个限制是写死的40个，查询Hive的最新的原代码，虽然判断的位置的提示信息已经变化，但是原理一样：

<!-- more -->

org.apache.hadoop.hive.ql.parse.VariableSubstitution：

```java
  public String substitute(HiveConf conf, String expr) {
    if (expr == null) {   
      return expr;
    }    
    if (HiveConf.getBoolVar(conf, ConfVars.HIVEVARIABLESUBSTITUTE)) {
      l4j.debug("Substitution is on: " + expr);
    } 
    else {     
      return expr;
    }    
    int depth = HiveConf.getIntVar(conf, ConfVars.HIVEVARIABLESUBSTITUTEDEPTH);    return substitute(conf, expr, depth);
  }123456789101112
```

如果开启hive.variable.substitute（默认开启），则使用SystemVariables的substitute方法和hive.variable.substitute.depth(默认为40)进行进一步的判断：

```java
  protected final String substitute(Configuration conf, String expr, int depth) {
    Matcher match = varPat.matcher("");
    String eval = expr;
    StringBuilder builder = new StringBuilder();    int s = 0;    for (; s <= depth; s++) {
      match.reset(eval);
      builder.setLength(0);      int prev = 0;      boolean found = false;      while (match.find(prev)) {
        String group = match.group();
        String var = group.substring(2, group.length() - 1); // remove ${ .. }
        String substitute = getSubstitute(conf, var);        if (substitute == null) {
          substitute = group;   // append as-is
        } else {
          found = true;
        }
        builder.append(eval.substring(prev, match.start())).append(substitute);
        prev = match.end();
      }      if (!found) {        return eval;
      }
      builder.append(eval.substring(prev));
      eval = builder.toString();
    }    if (s > depth) {      throw new IllegalStateException(          "Variable substitution depth is deeper than " + depth + " for expression " + expr);
    }    return eval;
  } 12345678910111213141516171819202122232425262728293031323334
```

如果使用的${}参数超过hive.variable.substitute.depth的数量，则直接抛出异常，所以我们在语句的前面直接加上set hive.variable.substitute.depth=100; 问题解决!

set命令的执行是在CommandProcessor实现类SetProcessor里具体执行，但是substitute语句同时也会在CompileProcessor中调用，也就是在hive语句编译时就调用了，所以oozie在使用时调用beeline执行语句时，compile阶段就报出异常。

但是为什么Hue直接执行这个语句时没有问题? 因为hue在执行hive时使用的是python开发的beeswax，而beeswax是自己直接处理了这些变量，使用变量实际的值替换变量后再提交给hive执行：

```python
def substitute_variables(input_data, substitutions):
  """
  Replaces variables with values from substitutions.
  """
  def f(value):
    if not isinstance(value, basestring):      return value

    new_value = Template(value).safe_substitute(substitutions)    if new_value != value:
      LOG.debug("Substituted %s -> %s" % (repr(value), repr(new_value)))    return new_value  return recursive_walk(f, input_data)
```