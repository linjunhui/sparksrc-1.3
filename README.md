# Apache Spark 1.3 源码阅读

Spark is a fast and general cluster computing system for Big Data. It provides
high-level APIs in Scala, Java, and Python, and an optimized engine that
supports general computation graphs for data analysis. It also supports a
rich set of higher-level tools including Spark SQL for SQL and structured
data processing, MLlib for machine learning, GraphX for graph processing,
and Spark Streaming for stream processing.

<http://spark.apache.org/>


## 在Yarn执行流程
执行示例:

    ./bin/spark-submit --class org.apache.spark.examples.SparkPi \
        --master yarn \
        --deploy-mode cluster \
        --driver-memory 4g \
        --executor-memory 2g \
        --executor-cores 1 \
        --queue thequeue \
        lib/spark-examples*.jar \
        10

spar-submit的内容

    # dirname除去非目录部分，dirname $0 到submit上级目录，.. 再到上级目录，最终获得spark目录
    SPARK_HOME="$(cd "`dirname "$0"`"/..; pwd)"    
    # disable randomized hash for string in Python 3.3+
    export PYTHONHASHSEED=0    
    # 执行脚本 $@所有的参数
    exec "$SPARK_HOME"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"

spark-class 脚本内容

    # 增加spark的安装目录到环境
    export SPARK_HOME="$(cd "`dirname "$0"`"/..; pwd)"
    # 执行 加载Spark环境的 脚本
    . "$SPARK_HOME"/bin/load-spark-env.sh
    
    # Find the java binary 配置java的环境
    if [ -n "${JAVA_HOME}" ]; then
      RUNNER="${JAVA_HOME}/bin/java"
    else
      if [ `command -v java` ]; then
        RUNNER="java"
      else
        echo "JAVA_HOME is not set" >&2
        exit 1
      fi
    fi
    
    # Find assembly jar
    SPARK_ASSEMBLY_JAR=
    if [ -f "$SPARK_HOME/RELEASE" ]; then
      ASSEMBLY_DIR="$SPARK_HOME/lib"
    else
      ASSEMBLY_DIR="$SPARK_HOME/assembly/target/scala-$SPARK_SCALA_VERSION"
    fi
    
    num_jars="$(ls -1 "$ASSEMBLY_DIR" | grep "^spark-assembly.*hadoop.*\.jar$" | wc -l)"
    if [ "$num_jars" -eq "0" -a -z "$SPARK_ASSEMBLY_JAR" ]; then
      echo "Failed to find Spark assembly in $ASSEMBLY_DIR." 1>&2
      echo "You need to build Spark before running this program." 1>&2
      exit 1
    fi
    ASSEMBLY_JARS="$(ls -1 "$ASSEMBLY_DIR" | grep "^spark-assembly.*hadoop.*\.jar$" || true)"
    if [ "$num_jars" -gt "1" ]; then
      echo "Found multiple Spark assembly jars in $ASSEMBLY_DIR:" 1>&2
      echo "$ASSEMBLY_JARS" 1>&2
      echo "Please remove all but one jar." 1>&2
      exit 1
    fi
    
    SPARK_ASSEMBLY_JAR="${ASSEMBLY_DIR}/${ASSEMBLY_JARS}"
    
    LAUNCH_CLASSPATH="$SPARK_ASSEMBLY_JAR"
    
    # Add the launcher build dir to the classpath if requested.
    if [ -n "$SPARK_PREPEND_CLASSES" ]; then
      LAUNCH_CLASSPATH="$SPARK_HOME/launcher/target/scala-$SPARK_SCALA_VERSION/classes:$LAUNCH_CLASSPATH"
    fi
    
    export _SPARK_ASSEMBLY="$SPARK_ASSEMBLY_JAR"
    
    # The launcher library will print arguments separated by a NULL character, to allow arguments with
    # characters that would be otherwise interpreted by the shell. Read that in a while loop, populating
    # an array that will be used to exec the final command.
    CMD=()
    while IFS= read -d '' -r ARG; do
      CMD+=("$ARG")
# java -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main org.apache.spark.deploy.SparkSubmit    
# $LAUNCH_CLASSPATH" 依赖类，执行org.apache.spark.launcher.Main(在spark/lib/spark-assembly-1.5.1-hadoop2.4.0.jar) 
# 将org.apache.spark.deploy.SparkSubmit和org.apache.spark.examples.SparkPi作为参数传入
    done < <("$RUNNER" -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main "$@")
    exec "${CMD[@]}"
    
load-spark-env.sh

    FWDIR="$(cd "`dirname "$0"`"/..; pwd)"
    # -z 长度为0即变量不存在就为 true
    if [ -z "$SPARK_ENV_LOADED" ]; then
    # SPARK_ENV_LOADED不存在就导入
      export SPARK_ENV_LOADED=1
    
      # Returns the parent of the directory this script lives in.
      parent_dir="$(cd "`dirname "$0"`"/..; pwd)"
    
      user_conf_dir="${SPARK_CONF_DIR:-"$parent_dir"/conf}"
        # -f 文件存在则为真
      if [ -f "${user_conf_dir}/spark-env.sh" ]; then
        # Promote all variable declarations to environment (exported) variables
        # -a 标识已修改的变量，以供输出至环境变量,spark-env.sh定义了变量还没输出到环境
        set -a
        # 导入 spark-env.sh中的变量到环境变量
        . "${user_conf_dir}/spark-env.sh"
        set +a
      fi
    fi
    
    # Setting SPARK_SCALA_VERSION if not already set.
    # 如果没有定义环境变量SPARK_SCALA_VERSION就定义，其实在spark-env.sh中已经定义
    if [ -z "$SPARK_SCALA_VERSION" ]; then
    
        ASSEMBLY_DIR2="$FWDIR/assembly/target/scala-2.11"
        ASSEMBLY_DIR1="$FWDIR/assembly/target/scala-2.10"
    
        if [[ -d "$ASSEMBLY_DIR2" && -d "$ASSEMBLY_DIR1" ]]; then
            echo -e "Presence of build for both scala versions(SCALA 2.10 and SCALA 2.11) detected." 1>&2
            echo -e 'Either clean one of them or, export SPARK_SCALA_VERSION=2.11 in spark-env.sh.' 1>&2
            exit 1
        fi
    
        if [ -d "$ASSEMBLY_DIR2" ]; then
            export SPARK_SCALA_VERSION="2.11"
        else
            export SPARK_SCALA_VERSION="2.10"
        fi
    fi
## Building Spark

Spark is built using [Apache Maven](http://maven.apache.org/).
To build Spark and its example programs, run:

    mvn -DskipTests clean package

(You do not need to do this if you downloaded a pre-built package.)
More detailed documentation is available from the project site, at
["Building Spark"](http://spark.apache.org/docs/latest/building-spark.html).

## Interactive Scala Shell

The easiest way to start using Spark is through the Scala shell:

    ./bin/spark-shell

Try the following command, which should return 1000:

    scala> sc.parallelize(1 to 1000).count()

## Interactive Python Shell

Alternatively, if you prefer Python, you can use the Python shell:

    ./bin/pyspark
    
And run the following command, which should also return 1000:

    >>> sc.parallelize(range(1000)).count()

## Example Programs

Spark also comes with several sample programs in the `examples` directory.
To run one of them, use `./bin/run-example <class> [params]`. For example:

    ./bin/run-example SparkPi

will run the Pi example locally.

You can set the MASTER environment variable when running examples to submit
examples to a cluster. This can be a mesos:// or spark:// URL, 
"yarn-cluster" or "yarn-client" to run on YARN, and "local" to run 
locally with one thread, or "local[N]" to run locally with N threads. You 
can also use an abbreviated class name if the class is in the `examples`
package. For instance:

    MASTER=spark://host:7077 ./bin/run-example SparkPi

Many of the example programs print usage help if no params are given.

## Running Tests

Testing first requires [building Spark](#building-spark). Once Spark is built, tests
can be run using:

    ./dev/run-tests

Please see the guidance on how to 
[run all automated tests](https://cwiki.apache.org/confluence/display/SPARK/Contributing+to+Spark#ContributingtoSpark-AutomatedTesting).

## A Note About Hadoop Versions

Spark uses the Hadoop core library to talk to HDFS and other Hadoop-supported
storage systems. Because the protocols have changed in different versions of
Hadoop, you must build Spark against the same version that your cluster runs.

Please refer to the build documentation at
["Specifying the Hadoop Version"](http://spark.apache.org/docs/latest/building-with-maven.html#specifying-the-hadoop-version)
for detailed guidance on building for a particular distribution of Hadoop, including
building for particular Hive and Hive Thriftserver distributions. See also
["Third Party Hadoop Distributions"](http://spark.apache.org/docs/latest/hadoop-third-party-distributions.html)
for guidance on building a Spark application that works with a particular
distribution.

## Configuration

Please refer to the [Configuration guide](http://spark.apache.org/docs/latest/configuration.html)
in the online documentation for an overview on how to configure Spark.
