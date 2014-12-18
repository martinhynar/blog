---
layout: post
title: "Clojure - injecting project.clj data into runtime"
abstract: "I wanted to print version of my application from runtime, and I wanted to have it defined on single place - project.clj."
tags: clojure cli
---

I was writting an application, and I wanted to print version of that application into output that it was generating.
Reason was clear, when someone is consuming that output and finds some problem, I want to know which version was used.

### Solution

It was not that easy as I supposed it will be and thanks to this problem, I learned some more features of [Leiningen](http://leiningen.org).
Learning by doing, that's it.

Unfortunatelly, it is not that strainghtforward, as I would expect. Usually, when you search for solution, you will find
advices to read `project.clj` as resource from jar. That will do, but will fail when you do `lein run` and there is no jar yet.

My solution is realized using custom item in `:prep-tasks`.

Now, how I did that...

#### Step 1 - Tell Leiningen where custom prep tasks are

To do this, you will need `.lein-classpath` in your project with this content

~~~
tasks
~~~
{: .language-clojure .code}

Also, it is good idea to include this file in git versioning, as it is usually excluded by general `.lein*` rule.

#### Step 2 - Provide your custom prep task

Now, create `tasks/leiningen/inject_parameters.clj` file in your project. It will have this content

~~~
(ns leiningen.inject-parameters)

(defn inject-parameters [project & args]
  (let [resource-file (format "%s/resources/injected.edn" (:root project))
        parameters (select-keys project [:version :name])]
    (spit resource-file parameters :append false)
    )
  )
~~~
{: .language-clojure .code}

Important thing here is that Leiningen's prep tasks receive `project` parameter which is map containing all keys
from `project.clj` and some others describing structure of the project. E.g. `:root` contains base folder of the project.
Armed with this map, I read those parameters of the project definition that I need (you might need some other).
And finally, write them to `resources/injected.edn` file. Having this file in `resources` folder means (by default) that
it will be packed into jar and available for consumption in runtime.

> You might have objection, that this is no different in comparison with reading `project.clj` in runtime, but the
*AHA* part is yet to come.

So, now we have custom prep task that reads project metadata and spits them to some file. But noone uses it...

#### Step 3 - Use your custom prep task

So, having simplified `project.clj` like this, focus on the `:prep-tasks` key

~~~
(defproject
  org.example/showme "1.2.3"
  :description "An example project"
  
  ;Here comes the importnat part for injection into runtime
  :prep-tasks ["inject-parameters" "javac" "compile"]
  )
~~~
{: .language-clojure .code}

Normally, if `:prep-tasks` is ommited, it has `["javac" "compile"]` as default values. By putting "inject-parameters"
as the first argument we force Leiningen to run custom task before any evaluation in the application runtime
(calling our main namespace). The important thing is that this will be done both when `lein run` and `lein jar` is used
which means, we will have our `injected.edn` file available in resource no matter how we run our project!

Reading this file in runtime is piece of cake. This way, you can pass whatever parameters you want to runtime, just change implementation of the custom task.
