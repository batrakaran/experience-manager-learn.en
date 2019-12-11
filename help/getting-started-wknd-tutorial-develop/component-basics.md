---
title: Getting Started with AEM Sites - Component Basics
description: Understand the underlying technology of an Adobe Experience Manager (AEM) Sites Component through a simple `HelloWorld` example. Topics of HTL, Sling Models, Client-side libraries and author dialogs are explored.
products: SG_EXPERIENCEMANAGER/6.5/SITES
products: SG_EXPERIENCEMANAGER/6.4/SITES
targetaudience: target-audience new
mini-toc-levels: 1
topic: development
index: y
---

# Component Basics {#component-basics}

Understand the underlying technology of an Adobe Experience Manager (AEM) Sites Component through a simple `HelloWorld` example.

## Prerequisites {#prerequisites}

Review the required tooling and instructions for setting up a [local development environment](overview.md#local-dev-environment). Ensure that you have a fresh instance of Adobe Experience Manager available locally and that no additional sample/demo packages have been installed (other than required Service Packs).

## Objective {#objective}

1. Learn the role of HTL templates and Sling Models to dynamically render HTML.
2. Understand how Dialogs are used to facilitate authoring of content.
3. Learn the very basics of Client-side libraries to include CSS and JavaScript to support a component.

## What you will build {#what-you-will-build}

In this chapter you will perform several modifications to a very simple `HelloWorld` component. In the process of making updates to the `HelloWorld` component you will learn about the key areas of AEM component development.

## Get the starter project {#starter-project}

This chapter builds upon a generic project generated by the [AEM Project Archetype](https://github.com/adobe/aem-project-archetype). Watch the below video and review the [prerequisites](#prerequisites) to get started!

> VIDEO

Open a new command line terminal and perform the following actions.

1. In an empty directory, clone the [aem-guides-wknd](https://github.com/adobe/aem-guides-wknd) repository:

    ```shell
    $ git clone git@github.com:adobe/aem-guides-wknd.git
    Cloning into 'aem-guides-wknd'...
    ```

    >[!NOTE]
    >
    > Optionally, you can download the [`component-basics/start`]( https://github.com/adobe/aem-guides-wknd/archive/component-basics/start.zip) branch directly.

2. Navigate into the `aem-guides-wknd` directory:

    ```shell
    $ cd aem-guides-wknd
    ```

3. Switch to the `component-basics/start` branch:

    ```shell
    $ git checkout component-basics/start
    Branch component-basics/start set up to track remote branch component-basics/start from origin.
    Switched to a new branch 'component-basics/start'
    ```

4. Build and deploy the project to a local instance of AEM with the following command:

    ```shell
    $ mvn -PautoInstallSinglePackage clean install

    [INFO] ------------------------------------------------------------------------
    [INFO] Reactor Summary for aem-guides-wknd 0.0.1-SNAPSHOT:
    [INFO] 
    [INFO] aem-guides-wknd .................................... SUCCESS [  0.394 s]
    [INFO] WKND Sites Project - Core .......................... SUCCESS [  7.299 s]
    [INFO] WKND Sites Project - UI Frontend ................... SUCCESS [ 31.938 s]
    [INFO] WKND Sites Project - Repository Structure Package .. SUCCESS [  0.736 s]
    [INFO] WKND Sites Project - UI apps ....................... SUCCESS [  4.025 s]
    [INFO] WKND Sites Project - UI content .................... SUCCESS [  1.447 s]
    [INFO] WKND Sites Project - All ........................... SUCCESS [  0.881 s]
    [INFO] WKND Sites Project - Integration Tests Bundles ..... SUCCESS [  1.052 s]
    [INFO] WKND Sites Project - Integration Tests Launcher .... SUCCESS [  1.239 s]
    [INFO] ------------------------------------------------------------------------
    [INFO] BUILD SUCCESS
    [INFO] ------------------------------------------------------------------------
    ```

5. Import the project into your preferred IDE by following the instructions to set up a [local development environment](overview.md#local-dev-environment).

## Component Authoring {#component-authoring}

Components can be though of as small modular building blocks of a web page. In order to re-use components, the components must be configurable. This is accomplished via the author dialog. Next we will author a simple component and inspect how values from the dialog are persisted in AEM.

Below are the high level steps performed in the above video.

1. Create a new page named **Component Basics** beneath **WKND Site** `>` **US** `>` **en**.
2. Add the **Hello World Component** to the newly created page.
3. Open the dialog for the component and enter some text. Save the changes to see the message displayed on the page.
4. Switch in to developer mode and view the Content Path in CRXDE-Lite and inspect the properties of the component instance.
5. Use CRXDE-Lite to view the `cq:dialog` and `helloworld.html` script located at `/apps/wknd/components/content/helloworld`.

## HTML Template Language Scripts {#htl-templates}

HTML Template Language or [HTL](https://docs.adobe.com/content/help/en/experience-manager-htl/using/getting-started/getting-started.html) is a light-weight, server-side templating language used by AEM components to render content.

Next we will update the `HelloWorld` component to render a new property named `message`.

Switch to the Eclipse IDE