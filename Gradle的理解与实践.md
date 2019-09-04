### 一、外部依赖（external dependencies）
```groovy
import org.apache.commons.codec.binary.Base64

buildscript {
    repositories {
        mavenCentral()
    }   

    dependencies {
        classpath group: 'commons-codec', name: 'commons-codec', version: '1.2'
    }   
}

task encode {
    doLast {
        def byte[] encodedString = new Base64().encode('hello world\n'.getBytes())
        println new String(encodedString)
    }   
}
```
执行: gradle -q encode
输出：aGVsbG8gd29ybGQK
### 二、自定义Plugin
```groovy
class GreetingPlugin implements Plugin<Project> {                                                                                 
    void apply(Project project) {
        project.task('haha') {
            doLast {
                println 'Hello from the GreetingPlugin'
            }   
        }   
    }   
}

// Apply the plugin
apply plugin: GreetingPlugin
```
执行: gradle -q hello
输出: Hello from the GreetingPlugin

### 三、使用扩展对象，使得Plugin可配置(configurable)
```groovy
class GreetingPluginExtension {                                                                                                   
    String message = 'Hello from GreetingPlugin'
}

class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        // Add the 'greeting' extension object
        def extension = project.extensions.create('greeting', GreetingPluginExtension)
        // Add a task that uses configuration from the extension object
        project.task('hello') {
            doLast {
                println extension.message
            }   
        }   
    }   
}

apply plugin: GreetingPlugin

// Configure the extension
greeting.message = 'Hi from Gradle'
```
执行: gradle -q hello
输出: Hi from Gradle
### 三、使用闭包对扩展对象（extension object）进行分组设置
GreetingPluginExtension是一个扩展对象(extension object)，通过属性名greeting添加至项目中。可以为每一个extension对象添加一个configuration block，来进行分组设置.
```
class GreetingPluginExtension {
    String message
    String greeter
}

class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        def extension = project.extensions.create('greeting', GreetingPluginExtension)
        println extension
        project.task('hello') {
            doLast {
                println "${extension.message} from ${extension.greeter}"
            }   
        }   
    }   
}

apply plugin: GreetingPlugin

// Configure the extension using a DSL block
greeting {
    message = 'Hello'
    greeter = 'Gradle'
}
```
执行: gradle -q hello
输出: Hello from Gradle

### 四、依赖
#### Lazy dependsOn
```groovy
task taskX(dependsOn: 'taskY') {
    doLast {
        println 'taskX'
    }   
}

task taskY {
    doLast {
        println 'taskY'
    }   
}
```
#### 计数
```groovy
task count {
    doLast {
        4.times {
             print "$it "
        }   
    }   
}
```
输出: 0 1 2 3

#### Dynamic Task
```
4.times { counter ->
    task "task$counter" {
        doLast {
            println "I'm task number $counter"
        }   
    }   
}
```

### 五、生命周期（build Lifecycle）
```
gradle.addBuildListener(new BuildListener() {
    @Override
    void buildStarted(Gradle gradle) {
        println("gradle.buildStarted")
    }

    //初始阶段, settings.gradle解析完成之后执行.
    @Override
    void settingsEvaluated(Settings settings) {
        println("gradle.settingsEvaluated")
    }

    //初始阶段, 在settingsEvaluated之后执行.
    @Override
    void projectsLoaded(Gradle gradle) {
        println("gradle.projectsLoaded")
    }

    //配置阶段, project evaluated完成后执行.
    @Override
    void projectsEvaluated(Gradle gradle) {
        println("gradle.projectsEvaluated")
    }

    //执行阶段, task执行完成后执行.
    @Override
    void buildFinished(BuildResult result) {
        println("gradle.buildFinished")
    }
})

//配置阶段, 每个project配置前都会执行.
gradle.beforeProject { project ->
    println("gradle.beforeProject: " + project)
}

gradle.afterProject { project, projectState ->
    if(projectState.failure){
        println "gradld.afterProject: " + project + " FAILED"
    } else {
        println "gradle.afterProject: " + project + " succeeded"
    }
}

gradle.allprojects(new Action<Project>() {
    @Override
    void execute(Project project) {
        //配置阶段,  每个任务evaluate之前都会执行, 在beforeProject之后.
        project.beforeEvaluate { project
            println "project.allprojects beforeEvaluate: " + project
        }

       //配置阶段, 每个人物evaluated之后都会执行, 在afterProject之后.
        project.afterEvaluate { pro ->
            println("project.allprojects afterEvaluate: " + pro)
        }
    }
})

gradle.taskGraph.addTaskExecutionListener(new TaskExecutionListener() {
    //执行阶段, 在task执行前执行.
    @Override
    void beforeExecute(Task task) {
        println "gradle.taskGraph beforeExecute: " + task
    }

    //执行阶段, 在task执行后执行.
    @Override
    void afterExecute(Task task, TaskState state) {
        println "gradle.taskGraph afterExecute: " + task
    }
})

gradle.taskGraph.addTaskExecutionGraphListener(new TaskExecutionGraphListener() {
    //配置阶段, 在project evaluated之后执行.
    @Override
    void graphPopulated(TaskExecutionGraph graph) {
        println("gradle.taskGraph.graphPopulated ")
    }
})

//配置阶段, 在graph popluated之后执行
gradle.taskGraph.whenReady { taskGrahp ->
    println("gradle.taskGraph.whenReady ")
}
```

###project.container(java.lang.Class<T> type) 
```groovy
class Book {
    final String name
    File sourceFile

    Book(String name) {
        this.name = name
    }
}

class DocumentationPlugin implements Plugin<Project> {
    void apply(Project project) {
        // Create a container of Book instances
        def booksContainer = project.container(Book)
        booksContainer.all {
            sourceFile = project.file("src/docs/$name")
            println "sourceFile: " + sourceFile
        }

        // Add the container as an extension object
        //为project.extension.books赋值之后, books(Closure)方法可以使用.
        project.extensions.books = booksContainer

        //为调用project.extension.tips赋值之后, tips(Closure)方法可以使用. 因为books和tips都指向booksContainer, 
        //所以它俩是一回事, 是指向同一个东西的不同别名.
        project.extensions.tips = booksContainer
    }
}

apply plugin: DocumentationPlugin

tips {
    quickTip {

    }

    userTip {
    }

}

// Configure the container
books {
        quickStart {
            sourceFile = file('src/docs/quick-start')
        }

        userGuide {
        }

        developerGuide {
        }
}

task booksTask {
    doLast {
        books.each { book ->
            println "$book.name -> $book.sourceFile"
        }

        println "---------------------------------"

        tips.each { tip ->
            println "$tip.name -> $tip.sourceFile"
        }        
    }
}
```
输出：(books和tips会输出两遍，两遍，两遍！)
developerGuide -> /Applications/test/src/docs/developerGuide
quickStart -> /Applications/test/src/docs/quick-start
quickTip -> /Applications/test/src/docs/quickTip
userGuide -> /Applications/test/src/docs/userGuide
userTip -> /Applications/test/src/docs/userTip
---------------------------------
developerGuide -> /Applications/test/src/docs/developerGuide
quickStart -> /Applications/test/src/docs/quick-start
quickTip -> /Applications/test/src/docs/quickTip
userGuide -> /Applications/test/src/docs/userGuide
userTip -> /Applications/test/src/docs/userTip

### 另一个例子：
```groovy
class TestDomain {
    //必须定义一个 name 属性，并且这个属性值初始化以后不要修改
    String name
    String msg

    //构造函数必须有一个 name 参数
    public TestDomain(String name) {
        this.name = name
    }

    void msg(String msg) {
        this.msg = msg
    }

    String toString() {
        return "name = ${name}, msg = ${msg}"
    }
}

//创建一个扩展
class TestDomainPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        //将test属性和TestTrackerExtension类关联起来.
        def testTrackerExtension = project.extensions.create("test", TestTrackerExtension)
        project.test.extensions.testDomain = project.container(TestDomain)

        project.task("hello") {
            doLast {
                println "task hello, testProperty: " + testTrackerExtension.testProperty
            }
        }
    }
}

class TestTrackerExtension {
    String testProperty
}

apply plugin: TestDomainPlugin

test {
    testProperty = "Haha"
    testDomain {
        domain2 {
            msg "This is domain2"
        }
        domain1 {
            msg "This is domain1"
        }
        domain3 {
            msg "This is domain3"
        }
    }   
}

task myTask << {
    test.testDomain.all { data ->
        println data //对应data.toString()        
    }
}
```
 命令：gradle -q hello
输出：
task hello, testProperty: Haha
命令：gradle -q myTask
输出：
name = domain1, msg = This is domain1
name = domain2, msg = This is domain2
name = domain3, msg = This is domain3
