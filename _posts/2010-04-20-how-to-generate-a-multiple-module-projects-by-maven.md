---
layout: post
title: How to generate a multiple module projects by maven
date: 2010-04-20 15:06:42.000000000 +08:00
categories:
- 技术
tags:
- Java
- Maven
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '1532653934'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
### **Project layout**

Due to the workspace idea many Eclipse users are used to a flat layout and therefore want to keep this structure. The following sample shows how to handle maven multiple module projects with Eclipse for both the standard maven hierachical project layout and the flat Eclipse-like layout.

#### Hierachical project layout

Suppose Eclipse is your favorite SCM client, this step by step example shows how to set up a new mutiple module project.

1. Set up a new Eclipse workspace called _step-by-step_ and add the _M2\_REPO_ classpath variable.
2. Open the command line shell and change to the newly created workspace directory.
3. From the command line, create a new maven project using the archetype plugin.

    mvn archetype:create -DgroupId=guide.ide.eclipse -DartifactId=guide-ide-eclipse

4. Create a new simple project _guide-ide-eclipse_ inside the _step-by-step_ workspace with Eclipse (From the menu bar, select File \>New \> Project. Select Simple \> Project). Eclipse will create a simple _.project_-file for your _guide-ide-eclipse_-project and you should be able to see the _pom.xml_-file.
5. Delete the _src_-folder and open the _pom.xml_-file to change the packaging of your parent project to _pom_.

      <packaging>pom</packaging>

6. From the command line change to the _guide-ide-eclipse_ project directory and create some modules.

    cd guide-ide-eclipse
    mvn archetype:create -DgroupId=guide.ide.eclipse -DartifactId=guide-ide-eclipse-site
    mvn archetype:create -DgroupId=guide.ide.eclipse.core -DartifactId=guide-ide-eclipse-core
    mvn archetype:create -DgroupId=guide.ide.eclipse.module1 -DartifactId=guide-ide-eclipse-module1

7. Add the newly created modules to your parent pom.

      <modules>
        <module>guide-ide-eclipse-site</module>
        <module>guide-ide-eclipse-core</module>
        <module>guide-ide-eclipse-module1</module>
      </modules>

8. Add the parent to the POMs of the new modules:

      <parent>
      <groupId>guide.ide.eclipse</groupId>
      <artifactId>guide-ide-eclipse</artifactId>
      <version>1.0-SNAPSHOT</version>
      </parent>

9. Add dependency from _module1_ to the _core_-module:

        <dependency>
          <groupId>guide.ide.eclipse.core</groupId>
          <artifactId>guide-ide-eclipse-core</artifactId>
          <version>1.0-SNAPSHOT</version>
        </dependency>

10. Install the project in your local repository and generate the Eclipse files:

    mvn install
    mvn eclipse:eclipse

11. Check in your project using the Eclipse team support (select from the context menu Team \> Share Project). _Note:_ Do not check in the generated Eclipse files. If you use CVS you should have a _.cvsignore_-file with the following entries for each module:

    target
    .classpath
    .project
    .wtpmodules

Even the parent project should have this _.cvsignore_-file. Eclipse will automatically generate a new simple _.project_-file when you check out the project from the repository.

From now on you have different options to proceed. If you are working on all modules simultanously and you'd rather have Eclipse project dependencies than binary dependencies, you should set up a new workspace and import all projects from _step-by-step/guide-ide-eclipse_. Note, you have to delete the _.project_-file of your parent project before. The result is the same as checking out the whole project from the command line, running _mvn eclipse:eclipse_ and finally importing the projects into your Eclipse workspace. In both cases you will be able to synchronize your changes using Eclipse.

In case of large projects with many developers involved, it can be tedious to check out all modules and keep them up to date. Especially if you are only interested in one or two modules. In this case using binary dependencies is much more comfortable. Just check out the modules you want to work on with Eclipse and run _mvn eclipse:eclipse_ for each module (see also [Maven as an external tool][0]. Of course, all referenced artifacts must be available from your maven repository.

#### 

It is possible to move the parent POM in its own directory on the same level with the referenced modules, thus resulting to a Flat Project Layout.

Using a Flat Project Layout you can checkout and edit the parent POM without checking out the whole project.

1. Create a new directory under _guide-ide-eclipse_ called _guide-ide-eclipse-project_ and move the parent POM to it.
2. Change the module references in the parent POM to:

      <modules>
        <module>../guide-ide-eclipse-site</module>
        <module>../guide-ide-eclipse-core</module>
        <module>../guide-ide-eclipse-module1</module>
      </modules>

**Issue:** The Maven Release Plugin does not support the flat structure ([MRELEASE-261][1]).

[0]: ./usage.html
[1]: http://jira.codehaus.org/browse/MRELEASE-261