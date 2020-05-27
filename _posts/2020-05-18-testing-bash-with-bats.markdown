---
layout: post
title:  "Testing Bash with BATS - Part 3"
author: "Andreas Heumaier"
date:   2020-05-18 11:01:07 +0200
categories: tech
---

The idea of programmers testing as they go made a come-back starting in the mid 1990s, although up to the present time the vast majority of programmers still don’t do it. Infrastructure engineers and system administrators test their scripts even less diligently than programmers test their application code.

As we move into an era where rapid deployment of complicated solutions comprising numerous autonomous components is becoming the norm, and “cloud” infrastuctures require us to manage thousands of come-and-go VMs and containers at a scale that can’t be managed using manual methods, the importance of executable, automated testing and checking throughout the development and delivery process can’t be ignored; not only for application programmers, but for everyone involved in IT work.

The [Bash Automated Testing System (BATS)](https://github.com/bats-core/bats-core) enables developers writing Bash scripts and libraries to apply the same practices used by other software developers to their Bash code.

Unit tests makes a great technique to check your code. In this tutorial, I will demonstrate how to write unit tests in Bash and you'll see how easy it is to get them going in your own project.

##  Tips for testing

A testing unit should focus on one tiny bit of functionality and prove it correct.

Each test unit must be fully independent. Each test must be able to run alone, and also within the test suite, regardless of the order that they are called. The implication of this rule is that each test must be loaded with a fresh dataset and may have to do some cleanup afterwards. This is usually handled by `setUp()` and `tearDown()` methods.

Try hard to make tests that run fast. If one single test needs more than a few milliseconds to run, development will be slowed down or the tests will not be run as often as is desirable. In some cases, tests can’t be fast because they need a complex data structure to work on, and this data structure must be loaded every time the test runs. Keep these heavier tests in a separate test suite that is run by some scheduled task, and run all other tests as often as needed.

Learn your tools and learn how to run a single test or a test case. Then, when developing a function inside a module, run this function’s tests frequently, ideally automatically when you save the code.
Always run the full test suite before a coding session, and run it again after. This will give you more confidence that you did not break anything in the rest of the code.
It is a good idea to implement a hook that runs all tests before pushing code to a shared repository.

If you are in the middle of a development session and have to interrupt your work, it is a good idea to write a broken unit test about what you want to develop next. When coming back to work, you will have a pointer to where you were and get back on track faster.

When something goes wrong or has to be changed, and if your code has a good set of tests, you or other maintainers will rely largely on the testing suite to fix the problem or modify a given behavior. Therefore the testing code will be read as much as or even more than the running code. A unit test whose purpose is unclear is not very helpful in this case.

Another use of the testing code is as an introduction to new developers. When someone will have to work on the code base, running and reading the related testing code is often the best thing that they can do to start. They will or should discover the hot spots, where most difficulties arise, and the corner cases. If they have to add some functionality, the first step should be to add a test to ensure that the new functionality is not already a working path that has not been plugged into the interface.

## Setting up our test environment

To start with, you’ll want to have a git repo set up with the basic outline of your project (a LICENSE, a file for your script, etc).
Next, you need some testing tools. 

Install [Bats](https://github.com/sstephenson/bats), [Bats-Support](https://github.com/ztombol/bats-support) and [Bats-Assert](https://github.com/ztombol/bats-assert) — I almost always install them as git submodules, so it’s easier for other contributors to get hold of them later.

[Bats](https://github.com/sstephenson/bats) is the core testing library, [Bats-Assert](https://github.com/ztombol/bats-assert) adds lots of extra asserts for more readable tests, and [Bats-Support](https://github.com/ztombol/bats-support) adds some better output formatting (and is a prerequisite for Bats-Assert).
To pull all these submodules into test/libs, run the below from the root of your git repo and commit the result:

{% highlight bash %}
mkdir -p test/libs

git submodule add https://github.com/sstephenson/bats test/libs/bats
git submodule add https://github.com/ztombol/bats-support test/libs/bats-support
git submodule add https://github.com/ztombol/bats-assert test/libs/bats-assert
{% endhighlight %}

## Organizing libraries and scripts for BATS coverage

Bash scripts and libraries must be organized in a way that efficiently exposes their inner workings to BATS. In general, library functions and shell scripts that run many commands when they are called or executed are not amenable to efficient BATS testing.

For example, [setup-linux.sh](https://github.com/aheumaier/bash-demos/blob/master/examples/setup-linux.sh) is a typical script that many people write. It is essentially a big pile of installer code. Some might even put this pile of code in a function in a library. But it's impossible to run a big pile of code in a BATS test and cover all possible types of failures it can encounter in separate test cases. **The only way to test this pile of code with sufficient coverage is to break it into many small, reusable, and, most importantly, independently testable functions**.

It's straightforward to add more functions to a library. An added benefit is that some of these functions can become surprisingly useful in their own right. Once you have broken your library function into lots of smaller functions, you can source the library in your BATS test and run the functions as you would any other command to test them.

Bash scripts must also be broken down into multiple functions, which the main part of the script should call when the script is executed. In addition, there is a very useful trick to make it much easier to test Bash scripts with BATS: Take all the code that is executed in the main part of the script and move it into a function, called something like run_main(). Then, add the following to the end of the script:

{% highlight bash %}
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]
then
  run_main
fi
{% endhighlight %}

This bit of extra code does something special. It makes the script behave differently when it is executed as a script than when it is brought into the environment with source. This trick enables the script to be tested the same way a library is tested, by sourcing it and testing the individual functions. For example, here is [setup_linux_refactor.sh](https://github.com/aheumaier/bash-demos/blob/master/examples/setup_linux_refactor.sh) refactored for better BATS testability.

## Writing and running tests
As mentioned above, BATS is a TAP-compliant testing framework with a syntax and output that will be familiar to those who have used other testing suites, such as JUnit or  RSpec. Its tests are organized into individual test scripts. Test scripts are organized into one or more descriptive @test blocks that describe the unit of the application being tested. Each @test block will run a series of commands that prepares the test environment, runs the command to be tested, and makes assertions about the exit and output of the tested command. Many assertion functions are imported with the bats, bats-assert, and bats-support libraries, which are loaded into the environment at the beginning of the BATS test script. Here is a typical BATS test block:

{% highlight bash %}
@test "test run_main should fail on missing env var" {
    NVIDIA_PKG="nvidia-diag-driver-local-repo-ubuntu1804-415.25_1.0-1_amd64.deb"
    NVIDIA_REPO="http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64"
    CUDA_PKG="cuda-repo-ubuntu1804_10.0.130-1_amd64.deb"
    USER_DEVSTACK=$(openssl rand -base64 12)
    unset PASS_DEVSTACK
    source ${profile_script}
    run run_main
    assert_failure 
    assert_output "Empty required env var found: var. ABORT"
}
{% endhighlight %}

If a BATS script includes setup and/or teardown functions, they are automatically executed by BATS before and after each test block runs. This makes it possible to create environment variables, test files, and do other things needed by one or all tests, then tear them down after each test runs. [setup_linux_refactor_test.bats](https://github.com/aheumaier/bash-demos/blob/master/examples/test/setup_linux_refactor_test.bats) is a full BATS test of our newly formatted [setup_linux_refactor.sh](https://github.com/aheumaier/bash-demos/blob/master/examples/setup_linux_refactor.sh) script. (The mock_docker command in this test will be explained below, in the section on mocking/stubbing.)

When the test script runs, BATS uses exec to run each @test block as a separate subprocess. This makes it possible to export environment variables and even functions in one @test without affecting other @tests or polluting your current shell session. The output of a test run is a standard format that can be understood by humans and parsed or manipulated programmatically by TAP consumers. Here is an example of the output for the CI_COMMIT_REF_SLUG test block when it fails:

{% highlight bash %}
 ✗ test install_nvidia_repos() should install nvidia_repos
   (from function `assert_failure' in file test/libs/bats-assert/src/assert.bash, line 140,
    in test file test/setup_linux_refactor_test.bats, line 49)
     `assert_failure' failed

   -- command succeeded, but it was expected to fail --
   output (25 lines):
     Calling docker_mock with wget -O /tmp/nvidia-diag-driver-local-repo-ubuntu1804-415.25_1.0-1_amd64.deb http://us.download.nvidia.com/tesla/415.25/nvidia-diag-driver-local-repo-ubuntu1804-415.25_1.0-1_amd64.deb
     --2020-05-19 11:17:11--  http://us.download.nvidia.com/tesla/415.25/nvidia-diag-driver-local-repo-ubuntu1804-415.25_1.0-1_amd64.deb
     Resolving us.download.nvidia.com (us.download.nvidia.com)... 192.229.221.58, 2606:2800:233:ef6:15dd:1ece:1d50:1e1
     (...)
   ** Did not delete , as test failed **

1 test, 1 failure
{% endhighlight %}

Here is the output of a successful test:

{% highlight bash %}
 ✓ test CONAN_USER_HOME is set
 ✓ test ORIG_PATH is set
 - test install_system_packages() should install provided packes (skipped)
 ✓ test install_nvidia_repos() should install nvidia_repos
 ✓ test install_cuda_repos() should install cuda_repos
 ✓ test run_main should fail on missin env var
 ✓ test run_main should be successfull

7 tests, 0 failures, 1 skipped
{% endhighlight %}

## Helpers

Like any shell script or library, BATS test scripts can include helper libraries to share common code across tests or enhance their capabilities. These helper libraries, such as bats-assert and bats-support, can even be tested with BATS.

Libraries can be placed in the same test directory as the BATS scripts or in the test/libs directory if the number of files in the test directory gets unwieldy. BATS provides the load function that takes a path to a Bash file relative to the script being tested (e.g., test, in our case) and sources that file. Files must end with the prefix .bash, but the path to the file passed to the load function can't include the prefix. build.bats loads the bats-assert and bats-support libraries, a small [helpers.bash](https://github.com/aheumaier/bash-demos/blob/master/examples/test/test_helper.bash) library, and a [docker_mock.bash](https://github.com/jasonkarns/bats-mock/tree/7e0fbf6bc705bd1b09daa2d5ff88962ddbe832f6) library (described below) with the following code placed at the beginning of the test script below the interpreter magic line:

{% highlight bash %}
#!/usr/bin/env bats
load 'libs/bats-support/load'
load 'libs/bats-assert/load'
load 'test_helper'
{% endhighlight %}

## Stubbing test input and mocking function calls

The majority of Bash scripts and libraries execute functions and/or executables when they run. Often they are programmed to behave in specific ways based on the exit status or output (stdout, stderr) of these functions or executables. To properly test these scripts, it is often necessary to make fake versions of these commands that are designed to behave in a specific way during a specific test, a process called "stubbing." It may also be necessary to spy on the program being tested to ensure it calls a specific command, or it calls a specific command with specific arguments, a process called "mocking."

The Bash shell provides tricks that can be used in your BATS test scripts to do mocking and stubbing. All require the use of the Bash export command with the -f flag to export a function that overrides the original function or executable. This must be done before the tested program is executed. Here is a simple example that overrides the cat executable:

{% highlight bash %}
function ls() { echo "THIS WOULD LS ${*}" }
export -f ls
{% endhighlight %}

This method overrides a function in the same manner. If a test needs to override a function within the script or library being tested, it is important to source the tested script or library before the function is stubbed or mocked. Otherwise, the stub/mock will be replaced with the actual function when the script is sourced. Also, make sure to stub/mock before you run the command you're testing. Here is an example from build.bats that mocks the raise function described in build.sh to ensure a specific error message is raised by the login fuction:

{% highlight bash %}
@test "test run_main should be successfull" {
    source ${profile_script}
    NVIDIA_PKG="nvidia-diag-driver-local-repo-ubuntu1804-415.25_1.0-1_amd64.deb"
    NVIDIA_REPO="http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64"
    CUDA_PKG="cuda-repo-ubuntu1804_10.0.130-1_amd64.deb"
    USER_DEVSTACK=$(openssl rand -base64 12)
    PASS_DEVSTACK=$(openssl rand -base64 12)
    VERSION_TOOLCHAIN=$(openssl rand -base64 12)
    LD_LIBRARY_PATH="/usr/local/lib"
    function install_nvidia_repos() { echo "This would install_nvidia_repos ${*}"; }
    export -f install_nvidia_repos
    function install_cuda_repos() { echo "This would install_cuda_repos ${*}"; }
    export -f install_cuda_repos
    function install_system_packages() { echo "This would install_system_packages ${*}"; }
    export -f install_system_packages
    function install_nvidia_drivers() { echo "This would install_nvidia_drivers ${*}"; }
    export -f install_nvidia_drivers
    function setup_xrdp() { echo "This would setup_xrdp ${*}"; }
    export -f setup_xrdp
    function install_vtd() { echo "This would install_vtd ${*}"; }
    export -f install_vtd
    run run_main
    assert_success
}
{% endhighlight %}

Normally, it is not necessary to unset a stub/mock function after the test, since export only affects the current subprocess during the exec of the current @test block. However, it is possible to mock/stub commands (e.g. cat, sed, etc.) that the BATS assert* functions use internally. These mock/stub functions must be unset before these assert commands are run, or they will not work properly.

Sometine it gest commplictaed if command alter stete even on the localal  machine like installin pasckages or modifiying package configurations. A great help is using `docker exec` command as mocking the underlying system: 

{% highlight bash %}
@test "test install_system_packages() should install provided packes" {
    source ${profile_script}
    function sudo() { docker_mock "${*}";  }
    export -f sudo 
    declare -ar PACKAGES=( wget curl gnupg2 xrdp libusb-0.1-4 libxvidcore4 libaa1 libfaad2 libxss1 libopencore-amrnb0 libopencore-amrwb0 )
    run install_system_packages
    assert_success
}
{% endhighlight %}

with [docker_mock](https://github.com/aheumaier/bash-demos/blob/master/examples/test/test_helper.bash) defined as:
{% highlight bash %}
docker_mock(){
    local command=$1
    local parameters="${@:2}"
    echo "Calling docker_mock with ${command} ${parameters}"
    docker exec -it -e DEBIAN_FRONTEND=noninteractive $DOCKER_SUT_ID bash -c "${command} ${parameters}"
    return $?
}
{% endhighlight %}

In general, scripts and libraries that meet one or more of the following should be tested with BATS:

- They are worthy of being stored in source control
- They are used in critical processes and relied upon to run consistently for a long period of time
- They need to be modified periodically to add/remove/modify their function
- They are used by others

Once the decision is made to apply a testing discipline to one or more Bash scripts or libraries, BATS provides the comprehensive testing features that are available in other software development environments.