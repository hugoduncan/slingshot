slingshot
=========

Enhanced throw and catch for Clojure
------------------------------------

  Provides `try+` and `throw+`. Each is 100% compatible with Clojure
  and Java's native `try` and `throw` both in source code and at
  runtime. Each also provides new capabilities intended to improve
  ease of use by leveraging Clojure's features like maps, records, and
  destructuring.

  Clojure's native `try` and `throw` behave much like those of Java:
  throw can accept objects derived from java.lang.Throwable and `try`
  selects from among catch clauses based on the class of the thrown
  object.

  In addition to fully supporting those uses (whether they originate
  from Clojure code or from Java code via interop), `try+` and
  `throw+` provide these enhanced capabilities:

  - `throw+` can throw any Java object, not just those whose class is
    derived from `java.lang.Throwable`.

    Clojure maps or records become an easy way to represent custom
    exceptions without requiring `gen-class`.

  - `catch` clauses within `try+` can catch any Java object thrown by
    `throw+`, Clojure's `throw`, or Java's `throw`. The first catch
    clause whose **selector** matches the thrown object will execute.

    a selector can be:

    - a **class name**: (e.g., `RuntimeException`, `my.clojure.record`),
      matches any instance of that class, or

    - a **key-values**: (e.g., `[key val & kvs]`), matches objects
      where `(and (= (get object key) val ...))`, or

    - a **predicate**: (function of one argument like `map?`, `set?`),
      matches any Object for which the predicate returns a truthy
      value, or

    - a **selector form**: a form containing one or more instances of
      `%` to be replaced by the thrown object, matches any object for
      which the form evaluates to truthy.

    - the class name, key-values, and predicate selectors are
      shorthand for these selector forms:

          `<class name>  => (instance? <class name> %)`

          `[<key> <val> & <kvs>] => (and (= (get % <key>) <val>) ...)`

          `<predicate>   => (<predicate> %)`

  - the binding to the caught exception in a catch clause is not
    required to be a simple symbol. It is subject to destructuring so
    the body of the catch clause can use the contents of a thrown
    collection easily.

  - in a catch clause, the context at the throw site is accessible via
    the hidden argument `&throw-context`.

  - `&throw-context` is a map containing:

    for Throwable caught objects:

        :object       the caught object;
        :message      the message, from .getMessage;
        :cause        the cause, from .getCause;
        :stack-trace  the stack trace, from .getStackTrace;
        :throwable    the caught object;

    for non-Throwable caught objects:

        :object       the caught object;
        :message      the message, from the optional argument(s) to throw+;
        :cause        the cause, captured by throw+, see below;
        :stack-trace  the stack trace, captured by throw+;
        :environment  a map from names to values for locals visible at
                      the throw+ site.
        :wrapper      the Throwable wrapper that carried the object
        :throwable    the outermost Throwable whose cause chain contains
                      the wrapper, see below;

  To throw a non-`Throwable` object, `throw+` wraps it in a
  `Throwable` wrapper. The wrapper is available via the `:wrapper`
  key in `&throw-context`.

  Between being thrown and caught, the wrapper may be wrapped by other
  exceptions (e.g., instances of `RuntimeException` or
  `java.util.concurrent.ExecutionException`). `try+` sees through all
  such wrappers to find the thrown object. The outermost wrapper is
  available within a catch clause a via the `:throwable` key in
  `&throw-context`.

  When `throw+` throws a non-`Throwable` object from within a `try+`
  catch clause, the outermost wrapper of the caught object being
  processed is captured as the cause of the new throw+.

Usage
-----

  project.clj

        [slingshot "0.10.2"]

  tensor/parse.clj

        (ns tensor.parse
          (:use [slingshot.slingshot :only [throw+]]))

        (defn parse-tree [tree hint]
          (if (bad-tree? tree)
            (throw+ {:type ::bad-tree :tree tree :hint hint})
            (parse-good-tree tree hint)))

  math/expression.clj

        (ns math.expression
          (:require [tensor.parse]
                    [clojure.tools.logging :as log])
          (:use [slingshot.slingshot :only [throw+ try+]]))

        (defn read-file [file]
          (try+
            [...]
            (tensor.parse/parse-tree tree)
            [...]
            (catch [:type :tensor.parse/bad-tree] {:keys [tree hint]}
              (log/error "failed to parse tensor" tree "with hint" hint)
              (throw+))
            (catch Object _
              (log/error (:throwable &throw-context) "unexpected error")
              (throw+))))

Credits
-------

  Based on clojure.contrib.condition, data-conveying-exception,
  discussions on the clojure mailing list and wiki and discussions and
  implementations by Steve Gilardi, Phil Hagelberg, and Kevin Downey.

License
-------

  Copyright &copy; 2011 Stephen C. Gilardi, Kevin Downey, and Phil Hagelberg

  Distributed under the Eclipse Public License, the same as Clojure.
