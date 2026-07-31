+++
draft = false
date = "2026-07-31T14:20:22+02:00"
title = "Reduce Visibility"
description = "How using DAGP and a custom IntelliJ plugin you can make a lot of your kotlin classes internal"
slug = ""
authors = []
tags = ["kotlin"]
categories = []
externalLink = ""
series = []
+++

## Reduce visibility
I like to use and build tools that let me improve the source code. I'm not talking about LLMs; I'm talking about deterministic tools that I can run over and over again on a source code.

Two weeks ago, an answer in the detekt Slack channel caught my attention. [Eugen Martynov](https://androiddev.social/@eugen) [commented on an interesting feature](https://kotlinlang.slack.com/archives/C88E12QH4/p1784297367396369?thread_ts=1784067418.051549&cid=C88E12QH4) in [Dependency Analysis Gradle Plugin (DAGP)](https://github.com/autonomousapps/dependency-analysis-android-gradle-plugin/). DAGP has an undocumented task called `publicTypeUsage` that lists all the `public` classes that can be `internal`. That's so good! Reducing visibility helps your source code in different ways. The one that I like the most: it helps to detect (and remove) dead code.

We already use DAGP at [Lidl Plus](https://play.google.com/store/apps/details?id=com.lidl.eci.lidlplus) to help us clean our dependencies weekly. `publicTypeUsage` was really easy to run. The task is slow, but in the end, there it was: a list of fully qualified classes that could be `internal`. The list was huge! There was no way I was going to edit all those classes by hand.

To do this kind of task, I usually create a Bash script using primarily `find`/`rg` (ripgrep) and `sed`. But in this case, I needed something a bit more advanced: I needed to work with a tool that knows Kotlin syntax. Luckily, some months ago, [Javier Tomás](https://github.com/inorichi) showed me an easy way to accomplish that: writing my own IntelliJ plugins. And a way to run them easily: [Live Plugin](https://plugins.jetbrains.com/plugin/7282-liveplugin).

### The plugin

I don't write these mini tools to last. The tool is an enabler to get something done and I don't plan to execute this regularly. For that reason, the code that you are going to see is not abstract or clean. It just does the job.

```kotlin
fun reduceVisibility(project: Project) {
    val declarations = ReadAction.computeBlocking<List<KtDeclaration>, Throwable> {
        File(project.basePath, "dagp-output.txt").readLines()
            .filter { it.startsWith("- ") }
            .map { it.removePrefix("- ") }
            .filterNot { it.startsWith("metro.hints.") } // 1
            .flatMap { name ->
                val scope = GlobalSearchScope.projectScope(project)
                KotlinFullClassNameIndex[name.replace("$", "."), project, scope].singleOrNull() // 2
                    ?.let { listOf(it) }
                    ?: run {
                        when {
                            name.endsWith("Kt") -> { // 3
                                val psiClass = JavaPsiFacade.getInstance(project)
                                    .findClass(name, scope) ?: error("JavaPsiFacade $name")

                                psiClass as KtLightClassForFacade
                                psiClass.files.flatMap { ktFile ->
                                    ktFile.declarations
                                        .filter { it is KtNamedFunction || it is KtProperty }
                                }
                            }
                            name.endsWith(".BuildConfig") -> emptyList()
                            else -> error(name) // 4
                        }
                    }
            }
            .filter { it.visibilityModifierTypeOrDefault() == KtTokens.PUBLIC_KEYWORD }
    }

    ApplicationManager.getApplication().invokeLater {
        WriteCommandAction.runWriteCommandAction(project) {
            declarations.forEach { it.addModifier(KtTokens.INTERNAL_KEYWORD) }
        }
    }
}
```

This plugin has two steps: first get the references to the declarations that can be `internal` and then apply the `internal` modifier.

#### Finding the declarations

I have the output of `publicTypeUsage` inside a file called `dagp-output.txt` at the root of the project[^1]. Basically, I run `./gradlew publicTypeUsage >dagp-output.txt`.

DAGP reports generated classes too. So we need to filter them out. The first filtering happens at the beginning. The classes that start with `metro.hints.` aren't going to exist in the code base, so I just ignore those (1). Then I look for the class in the project scope (2). Two different things can happen: either it exists (happy path, we are done) or it doesn't. In this latter case we need to check why it doesn't exist. If the class ends with `Kt`, it is very likely that it is a "synthetic" class created by Kotlin to add the top-level functions and properties (3). So we look for those and add them to the list. If it doesn't end with `Kt`, then I check if the name follows a common pattern of a generated class. For example `BuildConfig`. I like to be exhaustive so in the `else` block (4) I raise an exception. If there is a class that the code doesn't find, and it's not filtered out, I want an error so I can check what's going on. Usually, it is a missing filter, but you never know.

The code that I pasted is a simplification. The real one has a lot more filters because we use a lot of code generation.

The last point is to just get the declarations that are public. Otherwise, we could change some declarations from `private` to `internal` and we don't want that.

#### Making them internal
This is the easiest part: `declaration.addModifier(KtTokens.INTERNAL_KEYWORD)`.

#### Registering the action

The last missing piece: register the action. This action can be executed by name inside the IDE by pressing `Shift` twice and writing "Reduce visibility".

```kotlin
registerAction("Reduce visibility") { event: AnActionEvent ->
    val project = event.project!!
    ProgressManager.getInstance().run(
        object : Task.Backgroundable(project, "Reduce visibility", false) {
            override fun run(indicator: ProgressIndicator) {
                if (!project.isDisposed) reduceVisibility(project)
            }
        }
    )
}
```

### The results
Around 2k declarations moved from `public` to `internal`. We are really happy with the results!

Note: At least in our case there were some classes reported by DAGP that could not be internal[^2]. And we also found that there were classes that could be `internal` that were not in the list[^3].

[^1]: This could be done way nicer. DAGP generates a `JSON` with this information that I could parse. And I could even make the task run `publicTypeUsage` for me and then read that `JSON`. But as I said: fast and dirty.

[^2]: Especially classes related to [Metro](https://zacsweers.github.io/metro) but not only.

[^3]: I need to dig more into these false positives and false negatives to report the issues with reproducers.
