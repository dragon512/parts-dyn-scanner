
# Dynamic scanner example

This repo contains a build example of building curl via using a builder added to Parts to build a component via calling the components CMake or Automake build files. It shows off an example of a target scanner that will read the output of a directory and then call various builders to allow items to be installed correctly as well as have paths needed for dependent components.

This sample work in Parts was inspired by https://github.com/SCons/scons/wiki/DynamicSourceGenerator. At the more is used the same idea of calling builders in a scanner. It differs in that this example cannot be made to scale to large build correctly and general do rebuilds correctly. There are a number of reasons for this which I will go over more below. The main issue with this sample is having two or more of these builder depending on output of the others and having scons rebuild correctly. As I expanded the on the example to define a CMake builder I found a number of issues in SCons a quick summary of these that I will go in more detail below are:

1) Value cannot be used as targets in builders
2) Directories cannot be used as targets in builders
3) on rebuilds a recusive check for changed nodes is needed ( part of why chaining more than on builder using dynamic scanning logic can fail)
4) Scanner can be incorrectly called causing bad state to be passed
   1) builder can be caused more than once. ( The DynamicSourceGenerator tries to get around this via catching an exception, however in practice I found this just put bad state in larger builds causing problems as one builder was called but another way not)
   2) partial or out of date information was passed in to the builders on the first call to the scanner on a rebuild
   3) environment values defined in leaf dependent scanner had not been called yet leading to incorrect states when parent builder scanner was called on rebuilds
5) SCons does not handle Alias node as Sources well on rebuilds
6) The sample does not make a unque name for the Alias. Which means calling the builder twice will have issues as the node tree will be incorrect. This will need to a number of problem. The most common is a circular dependency in the node tree

For these reasons, there was a need for Parts to override and extend logic on core items in SCons to allow dynamic scanners to work correctly. This is also why it is not really possible to make a native SCons example is not possible or least not as easy to make as a Parts example.

## How to build

To build this one has to `pip install scons-parts` as well as a few tools. To help with this there is a vscode remote docker setup included to make it easy to build on Windows or Mac. There is also a pipenv PipFile to help with installing a virtual environment.

The build will report missing packages for Ubuntu and or Fedora. Red Hat and CentOS 7 or better should work as well but will need devtools to build as cmake 3.0 or better is needed.

Given the need packages, tools and compilers are installed correctly one can build everything via saying `scons all` however adding a `-j4` option will speed up the overall build greatly.

### Using pipenv

If you are on linux and have python3 and pipenv installed. Setting up a virtual enviroment can be as easy as:

```bash
pipenv install --three
pipenv shell
```

If you want to use setup your own enviroment run:

`pip install scons-parts`

to install Parts and SCons on the system. See below about needed system packages to allow the build to work.

### Using VS code remote

If you are using the [VS code](https://code.visualstudio.com/) and you have installed the [remote-container plugin](https://code.visualstudio.com/docs/remote/)containers there is a configuration to allow this to build inside a container. If using these options and you are on windows (and maybe mac) you will want to add the build via saying:

`scons all --cfg-file=docker.cfg`

which will do much of the build in the docker container avoiding issues that will break automake builds because of file permission issues. Keep in mind there is some overhead and depending on the Windows or Mac configuration of docker and possible antivirus software build can be noticeably slow vs building this Linux natively. I found a rebuild issues that may be related to a timestamp check being used in the current drop of Parts. This seems to only happen on docker as a native linux build will not show this.

### System packages needed

I might have a few extra packages here. I was reducing this from a larger build case I use. Please note Patchelf is needed to help with building the RPMs. the latest version `0.10` works best. You might have to build this for source as most distros have older version of this. If you need to build it this will allow you to build and add to your system quickly:

``` bash
git clone https://github.com/NixOS/patchelf.git \
&& cd patchelf \
&& ./bootstrap.sh \
&& ./configure \
&& make \
&& sudo make install \
&& cd .. \
&& rm -rf patchelf \
```

#### Ubuntu

This is just because you might like ubuntu over the others. You can build RPMs on it that would work on ubuntu however you probably want to make dpkg instead. This sample does not do this at this time because I have to update the builder for this in Parts and I have not done this yet.

`sudo apt install rpm cmake xsltproc libexpat1-dev docbook-xsl git autoconf automake libtool bison flex autotools-dev g++ g++-multilib libpcre3-dev libcap-dev libhwloc-dev libncurses5-dev zlib1g-dev libsigsegv2 libc-ares-dev libextutils-makemaker-cpanfile-perl libidn11-dev libjson-c-dev libcppunit-dev libreadline-dev patchelf`

#### Fedora

This should work as well for Centos and RHEL system however their may be small package differences.

`sudo dnf install git python3 make gcc g++ rpm rpm-build cmake expat-devel docbook-style-xsl libxslt-devel autoconf automake libtool bison flex libcap-devel pcre-devel hwloc-devel ncurses-devel zlib-devel c-ares-devel perl-ExtUtils-MakeMaker perl-Pod-Html-1.24-440 json-c-devel readline-devel patchelf`

Other Distros will need to install that systems package versions of the above.

### Build Targets

Parts defines a number of default targets. I am not going to go into details here but will provide some quick examples of values you can use.

 * **all** - will build everything
 * **brotli** will build brotli component and anything it depends on
 * **brotli.rpm** will build brotli rpm component and anything it depends on
 * **brotli::** will build brotli and any subparts. in this example case this is the same as **brotli.rpm**

Target values that Parts should accept are documented in a PDF [here:](http://parts.tigris.org/doc/PartsUserGuide.pdf) however this is old and out of date. The logic for targets has not changed. I have started to rewrite this in Sphinx. It will be a while before I get this finished and updated to the current drop of Parts.

### Notes on the structure of the sample

This sample defines a part that builds a given component and a sub-part that define build the package rpm for that component. This may not be the best strategy in a large system to do. It may be better to provide a packaging part file as a separate part instead of a sub-part to include in the SConstruct. This would make it easier for other users to decide and control if RPMs or some other package type is built or not.

# Details

This sample is to help show how a scanner can be used to define items to build dynamically in SCons. I find this can be very powerful and useful for certain cases such as calling documentation tools as part of a build. There are a few items that one as to be aware of that can cause issues with build and rebuilding correctly. I hope to attempt to outline these items here to help improve core SCons and usage of SCons with or without Parts extension.

## The Scanner cases I am showing here

This sample shows two ( well three) builder that uses a dynamic scanner in different ways. Two of them use a Directory SCanner. These are the CMake and AutoMake builder that call cmake or automake/configure projects to build and make an internal install and then call the scanner to define what needs to be installed and shared. For this sample, I redirected the `make install` output to a _destdir location to make it easier to look at. The default location under the _build directory is a longer path and harder to find. There is another issue with VariantDir in Scons that can cause issues make it needed to do this redirection because this sample has the build files defined out of the repo the holds the source. This why the value of `#` is used in some of the paths. Given a possible fix to the existing ( and once documented) `src_dir` keyword in Sconscript() I could address this quirk via alining the build file to think the "src_dir" is the source we checked out.
The general flow of these these builder is to make a chain of two builders. The first builder task is to call a set of actions to create a Makefile. The second builder is to make the directory that the makefile will install items to as scan that area to correctly install and sdk items that need to be shared with other components. In the simplest for the look like something this:

```python
makefile= env.Command('makefile','Configure')
outdirectory = env.Command(
    env.Directory("destdir"),
    makefile,
    target_scanner=env.ScanDirectory(
        env.Directory("destdir") # directory to scan
    )
```

There are a lot more details to deal with however, but this is the general flow. The source for the different builder and scanner can be found here:

* AutoMake() - https://bitbucket.org/sconsparts/parts/src/master/src/parts/pieces/automake.py
* CMake() = https://bitbucket.org/sconsparts/parts/src/master/src/parts/pieces/cmake.py
* ScanDirectory() - https://bitbucket.org/sconsparts/parts/src/master/src/parts/pieces/directoryscan.py

The other uses a custom scanner to help generate an RPM-based on files that are mapped to a given group. This case will read a file to define what files should be added to the rpm.spec file and added to the tar.gz file rpmbuild wants to see.

## Overview of Scanners

Scons defines a unique item that is not generally found in other build systems. This is the notion of a scanner. To understand the value of the scanner it is important to go over some basic quickly. SCons as a make replacement define the notion of rules in terms of builders. A builder is a combination of target outputs required sources to create the target and a set of one or more action that needs to run to build the target. Scons add the ability to define a scanner that allows for a generic way to dynamically add implicit dependencies need to correctly build the target. In scons the default scanners that are provided are generally a regular expression scanner that may look at some variable such as CPPATH or LIBPATH to help set correct dependencies for build a source file or linking a binary. SCons allow scanners to be added at the builder or a more global file extension level to help make it easier to auto-generate the addition of the dependencies without direct user interactions that are often common in other build systems or build generators. This one feature of Scons, I believe, sets SCons above other build system or build generators that exists today.
One of the side effects of the scanner is the ability to call other builders to build new items that become a dependency of the node the scanner is scanning. This allows adding whole new node trees into the build tree. This can very useful for cases in which:

* we only know a directory will be made, we don't know the exact names of what will exist in there but need to take actions based on what would exist. This is common for documentation tools such as sphinx or Doxygen and cases of calling a third party build system such as CMake. In this case, after the target is built we can scan for items of interest and call builders that add a new node tree that Scons will process.
* We need to define extra actions that are only known after other items have been built. This is common in cases of packaging systems that often have a file defining what files to add to the package and require a tar/zip file to be generated based on the generated definition file.

Scanners can be applied to the source nodes of a builder or the target node of a builder. The source case is common and easy to understand as these are cases such as looking for header files needed for a source file. Scanning target nodes are not as obvious and I feel benefit most from dynamic scanning logic.

### Issues with scanning calling builders

There are issues with doing this that can cause several problems when building in SCons. Issues such as:

1) Correctly chaining of builders using dynamic scanners can be tricky to get correct until one understands how scanners are called.
2) Having a builder being called twice.
3) Having incomplete data before calling a builder in the scanner

These issues can easily cause build failure, false rebuilding, and worse bad builds when rebuilding.

There are also issues in how SCons deals with Values and Directory Node as build targets. I have submitted issues on this and have made some monkey patches in Parts to deal with this. Hopefully, these will be applied to the core SCons logic after some review a tweaking to make sure everything is 100% correct.

### When are scanners called

The Taskmaster current can call a scanner up to three different times. In general, the logic is:

1) Call the scanner to see what extra nodes SCons needs to check for being out of date
2) Call again before it will try to build a target if any of the sources are out of date. If new sources are added here these are checked. However, SCons will not call the scanner again before building the target. Also if none of the sources or files returned by the scanner are out of date the scanner will not be called again.
3) After the target is built the node is scanned again and this information will be stored in the SCons DB. This case tends to make a lot of sense for target scanners as the target normally does not exist until after it has been built.

The order in which the scanners are called will be the target scanner first, then the source scanner. I tend to find one quirk in the current taskmaster logic is that with case 2) that scanner will not be called again preventing the scanner from adding needed files on a scanner pass (because it might not exist or the environment state is incomplete) will cause the builder to rebuild because the dependent sources changed but the scanner will not be called again before the build. This leads to bad builds, false rebuilds or ping-ponging in SCons always feeling it needs to rebuild because one pass was missing a file and the next pass adding it back in so scons feels it is always out of date. It should also be noted that the taskmaster has explicit logic to not loop on the scanner calls. I think there could be some improvements here, however, the logic that exists prevents an infinite loop that could happen if a scanner returns a different set of files every time it is called.

To understand this better why rebuilds have this to happen in the scanner I have found has to do with two general issues:

 1) Environment variables may not be fully resolved.
 2) Chain builders such as SharedLibrary inject levels of checks that allow items to look up to date

The first case can happen in a few different ways. The main issue is that a path or value is not added until another scanner is called. This is very easy to understand once we have widespread use of scanners calling builders. The taskmaster is going from top to bottom asking the question if a direct node is out of date or not. It calls a scanner on a header file to get dependents. This header file is generated from another dynamic scanner. It has not been called yet so at this point the path to the header the scanner would have added is not added yet to the environment of the target node. The default scanner fails to find the header and as such SCons remove it from the known children causing it to rebuild because "headerX.h is not a dependency anymore".
Another case that can happen here is the as the taskmaster is going down the tree a source file the scanner would call a builder on will internal make a chain of builder calls. For example, the SharedLibrary() call will map C and CPP source files to a SharedObject() call. What happens here is that the source file may have changed but results in an object file with the same MD5 CSIG. Since the object file has the same CSIG value the scanner is not called again as the directory node the taskmaster is checking look to be up-to-date. This prevents the scanner from being able to generate a correct list of dependency nodes because we want to add a value based on the source node however it is not up to date so it will not provide correct information. Once it is built the taskmaster is checking the object file, not the source file to decide if the scanner should be called again. This results in an incomplete set of nodes to being returned that will causes items to rebuild because a node is not a dependancy anymore, then it becomes a dependancy build after that, causing a rebuild again as the source node is up-to-date and does provide a correct set. This can get more problematic when trying to cache values as values stored in the cache may contain values from the last build, however, a node that existed in the previous build should not exist in the current build. Removing that state after it is defined is impossible once it is returned by a scanner call before a node is built.

#### Good practices for scanner and work around in Parts to deal with these issues

To deal with these issues I have found a few good general practices when using scanners.

##### Return only when you know all the sources are up-to-date

 I have found that a scanner should try to pass back values only when all the known sources are up-to-date. By up to date I mean all source nodes of the target and what these source nodes depend on, etc down to the leaf nodes. The current logic in SCons for checking if a node is changed only goes one level down and is heavily dependent on the taskmaster logic. This means that when SCons calls the changed() call it does not know if it is out of date. It only knows that at this time the current sources are out reported that they are different from the target point of view. If the changed call returns False, it might return True after the taskmaster can test all the source nodes and rebuild them. It can be viewed in SCons as not a problem as these node calls are not documented and are tied into the taskmaster logic which by design will go down the node tree and call this as it builds nodes. However, the way it does this assume the scanner are generally static in what would be returned. For example, it tends to assume a scan of a header file will always return the same values. However in practice what is returned is dependent on the compile-time value that may be defined as part of the build. This has not been seen as a major issue as of today as the scanner for headers is a little stupid and always returns all pattern the match "`#include ???`" and ignores header values it cannot map to disk.

To address this, Parts adds a function to answer the question if the node should be view as up-to-date based on all nodes in the tree that are below it. With a little caching and understanding of SCons has visited or built the node ( ie does it think it is up-to-date) there is little to no overhead for doing this. This allows the scanner to return values correctly. This does not solve the issue of calling a builder that may insert a chain of one or more extra builder that can result is the direct node looking up to date

##### Check up-to-date differently for depending on the source or target based scanners.
Given is the scanner if a source or target scanner how the check for the node to change has to be done differently.

###### Target case
 In the case of a target based scanner, it is best to check if the target explicit children node is up-to-date (children that are in sources or depends, not implicit). Parts added a has_children_changed() function to do this recursively. This ensures all nodes are checked and Environment values get resolved that may affect the scanner results. It is also wise to make sure the target scanner returns some other files that are consistent. For example, in the case of the DirectoryScanner in Parts this means returning values from another scanner such as the program scanner to ensure libraries needed dependancy are added. Just a quick note on this, the SCons ProgScanner only change the environment LIBS and LIBPATH values. It does not change the node itself, so if the node is a Dir, File or Value does not matter to it. Having this called in the case of a part calling CMake() builder, for example, makes sure that SCons will build the dependent actions ( ie other Parts) to build libraries that Parts CMake() builder may need to link the binaries correctly. Not doing this until the target is built or is viewed as up-to-date leads to rebuilds or build failures as the libraries are either missing or seen as a missing or added dependency on a rebuild.
###### Source case
In the case of a source node it is best to check that the node is up-to-date (Unlike the target case, this means all children, implicit and explicit). Parts added a has_changed() to deal with this correctly in the recursive case. Unlike the target case, the source has to be built and viewed as up-to-date before the target is built. So checking that the source is good is fine, while the target case is not built yet, so in that case, we want to make sure the explicit children are up-to-date.

##### Cache values
Given the scanner knows that the items it has are up-to-date, it is a good idea to cache these values and return them given the node is still viewed up-to-date. In the case of interactive mode, the node may become out of date. In this case, the scanner should clear its cache until the good state is known. This allows for at least one of the three possible calls to the scanner to be very fast, reducing overall build time.

##### Always have an explicit source node
This may be viewed more as good advice for builders in general. When a scanner is used in a builder that will call other builders SCons need to have an explicit source to check. Without this SCons taskmaster will not call a scanner a second time *ever*. This is because it does not see any sources that could ever be out of date. It also prevents being able to check if any children are out of date as a test to validate that the scanner can get a full and correct set of nodes to return. Not having an explicit source will cause an issue when this builder is changed with other build and will cause problems in many different rebuild cases. Having an anchor to allow SCons to go down the node tree and add a sub-tree in the middle with the scanner will work best and avoids many issues.

#### Scan scanner function quirks and other notes

One concern that can happen is calling a build twice. Doing this in SCons causes an error. In Parts, I have cases in which this can happen correctly. I have added an `allow_duplicate=True` flags that Parts apply to any builder to work around this in certain cases. It is generally not useful given you to have cached the good result correctly as the builder should only be called once. I should note that the `allow_duplicate=True` feature was added because of cases of cross-building in which a part would be built twice and need to install a common file to a shared location such as a configuration or a header file. The point is caching is harder than it looks and this feature was useful to get the full solution working. Also caching is a good thing when done correctly.

A thought here is that some of this work would be easier if I could call a builder again and say ignore the last results. When defining the RPM builder I have it might have been easier to cache the last know results on the first pass and on the second call I could replace that with the updated result. Currently, this logic fails because SCons does not like or have a way to call a builder twice in a way to remove defined sources. When rebuilding this can become an issue when you want to remove a file that would be called in the builder the scanner would call. One can rebuild everything from scratch to work around this, but we are busy and would rather only rebuild what is needed. That is why we are using SCons in the first place.

SCons scanners allow one to define a scan_check function. This function intends to pass a True/False value to tell SCons to call the full scanner or not. What I had hoped for in the logic was that if it returns a False Scons would call this function to try to call the function again till it returned True, or at least call it one more time. However, the logic is that the scanner just returns an empty list. I had hoped that I could do a better up-to-date check here and SCons would call the scanner again once it was known the state was good to call the scanner. Since SCons just view this as if the scanner returned an empty list it added more complexity than just doing the check-in the main scanner function which made it easier to return extra cached values. It could be useful to have SCons change the logic here to call the scanner again if the scan_check function returns False once the known sources are view as being built or up-to-date then error out or complain the scanner was skipped before continuing building the node.

As stated above, the scanner may be called once or twice before Scons will try to build the target depending on if SCons think the sources are up-to-date or not. I also made a note about an issue with a builder that makes a chain of builder calls internally. This lead to an interesting quirk I had with defining the Directory Scanner used for the AutoMake and CMake builders. To allow these scanners to work correctly I had to define a state file that would be seen as a side effect of the directory being built. This state file would have as explicit depends on the output of the builder I called in the scanner. This allows SCons to correctly rebuild items in the directory if they had been removed and allow dependencies to correctly be setup for chains builder calling this scanner ( ie think of a builder call Curl build system that depends on nghttp2 that depends on OpenSSL). The state file was a JSON file and because it contained a list of the binaries the file would generally have the same MD5 csig if it was rebuilt. SCons by design will not rebuild if the decider feels the direct depends are the same. This would cause issues as the build may have added an option that would have caused changes in the binaries, but the text file based state file would be viewed as the same. This would cause cases in scons would stop building because it did not look deep enough. I worked around this by adding a node-specific decider object to do a timestamp check instead. This generally worked great, however this lead to a different issue I found in the RPM builder. In this case, a spec file would not be regenerated correctly or at all for cases such as building a devel based RPM. The reason for this had to do with the builder for spec file would have a depends on the state file to get the file needed to spec file. This file would depend on some headers let say that would be copied to a staging area. Because these files looked the same at an MD5 csig layer the SCons would short circuit when calling the scanner for the RPM file. I fixed this by adding a check for the state files to be viewed as out of date is the source has a different timestamp. By default, the decider is done at the source level, not the target. The state file however needed to be out of date if the target was rebuilt, so a timestamp would be a correct choice here. THat almost fixed the issue until I SCons still did not rebuild correctly. The issue was that the timestamp was not updated even if I deleted the file to have it recopied. The issue here turned out to be the use of the python shutil.copy2() function in the copy actions define in SCons and Parts. I change the copy function in Parts to use shutil.copy() instead to correct the problem. For this case, I had added a better Decider function to say that this node uses the default time-stamp check, vs having to pass a function to reimplement the logic of doing a timestamp check. I think this logic is missing as an oversite. I also have to change the copy2 to copy() logic. I think copy is what we want for a build system. Understandably, this would not be noticed as an issue as SCons does by default the better signature check. Last of all I had a case in which I had these state files in Parts that are JSON files needing to be view as out of date if any of the sources had been rebuilt. This is an odd case from the common cases of how deciders work in SCons. However for cases of text files that have state in them being able to have them be viewed out of date because the source was built is needed. Using timestamp seems to be good enough as long as the timestamp logic can be applied at the target level. This is a case in which a source node may need to have a different decider depending on the target. We don't have an official way to say this in SCons.

## Other changes I had to make in Parts

The major changes I had to monkey patch with SCons or add to parts was:

1) adding the target scanner to Command(). I did this via adding a CComand builder to not conflict with SCons. I also have a PR (which I still believe is open as I needed to updated documentation for it)
2) add some overrides to Directory nodes to allow then to be used correctly with builders.

I could work around with not doing 1) however given the CMake() and AutoMake() builder are a chain of builder calls this was the fastest and quickest way to get this prototype and working. I know both of these builders should have more fixes made to them to make them more useful and more SCons "Tool" like.
I have issues opened showing issues with Value and Directory node note working as a target for a builder, or more correctly these nodes cannot have builder mapped to them currently. I was going to implement the state logic via the Value node originally. This would have an advantage of the state being stored in the .sconsign.dblite without making lots of files. I hope to improve the system this way. However to get this working using file node just worked better and was easier to debug as I did not need to dump DB state to the screen. The directory was needed for the directory scanner cases and needed me to address some missing logic. There where a few functions that had to be implemented but only exist on the File node class to would allow the scanner to be called correctly. There was also an issue in the Directory node state is not stored the .sconsign.dblite. There is no binfo for directory or value nodes. I added this information for the Dir node class and had to add a check on how the storage logic worked depending if the builder was one defined by the user or was the internal MkdirBuilder builder used by SCons by default to build the directory if it does not exist. This was not to hard to add and allows the Directory to be propper targets for a defined builder when Parts is being used. Hopefully, a cleaned-up version of the monkey patches I did here will be applied to SCons in the future

The dynamic scanner I feel is a powerful technique to use in SCons. It is an advanced technique to master correctly that requires some tweaks to the core SCons code to allow it to work correctly at scale.
