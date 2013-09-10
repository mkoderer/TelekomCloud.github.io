---
layout:   post
title:    "New Ways of Tempest Stress Testing"
date:     2013-09-05
author:   Marc Koderer
comments: true
---

# Overview

There are many stress test frameworks for OpenStack around that mainly do the very same. They following fixed scenarios and forking many of worker processes. Often they are hard to enhance since they are written for only one purpose.

With [blueprint stress-test](https://blueprints.launchpad.net/tempest/+spec/stress-tests) the community of Tempest developers tried to build up a single and very flexible stress test framework inside of Tempest.

# A Stress Test is not a Test Domain

In the past the stress tests had their own area inside of Tempest. If a new test was introduced it was mainly a clone of an exiting API, a scenario test or a mixture of both. But do stress tests really have their own testing domain? 

Two main purposes of stress test are obvious:

 * Having a framework to find/reproduce race conditions
 * Having a framework to simulate real-life load with a mixture of load profiles

But what stays is the fact that everything is already covered by existing Tempest tests. If we mix-up API test we can easily find race conditions, if we mix-up scenario test we can simulate load profiles.

# Tempest Stress Test Core

The core of the stress test framework is quite simple. It's responsible for forking worker processes and it will summarize the results of it. How many processes should be forked is configurable in a json configuration file which also can provide multiple argument for the stress test itself.

# Integration of Existing Tests

In order to stop duplication code we started to integrate existing Tempest tests inside the stress test framework. A wrapper was build to call any kind of unit test and make it available for the framework. With that it's easy to mix-up existing tests. A configuration of two unit test look like this:


    [{"action": "tempest.stress.actions.unit_test.UnitTest",
      "threads": 8,
      "kwargs": {"test_method": "tempest.cli.simple_read_only.test_glance.\
                 SimpleReadOnlyGlanceClientTest.test_glance_fake_action",
                 "class_setup_per": "process"},
      "action": "tempest.stress.actions.unit_test.UnitTest",
      "threads": 8,
      "kwargs": {"test_method": "tempest.api.volume.test_volumes_actions.\
                 VolumesActionsTest.test_attach_detach_volume_to_instance",
                 "class_setup_per": "process"},
    }]


So this will fork in total 16 processes that will do some glance and cinder testing.

## The Stress Test Discovery

The test discovery in general looks like this:

<img src="/images/2013-09-05-new-ways-of-tempest-stress-testing/tempest_stress_discovery.png" alt="Drawing" style="width: 650px;"/>

Instead of manually adding exiting tests inside the stress test framework the next step was to introduce a decorator where every test developer can decide whether he want to make the test available or not. To be more precise the question is does a certain test make sense to be used as a stress test. The decorator looks like this:


    @stresstest(class_setup_per='process')
    @attr(type='smoke')
    def test_attach_detach_volume_to_instance(self):

It can be simply added to any existing unit test or used as the only purpose for a test. Internally it uses the existing mechanism of attribute discovery of `testtools` and add a attribiute `stress`. There is one mandatory parameter `class_setup_per` since it must be decided when the `setUpClass` function should be called. For every process, for every action or just globally. This depends on the content of the `setUpClass` and must be decided by the developer. In many cases a call on process level is sufficient.

All defined test attributes like `smoke` or `gate` can be used as filter for the discovery function.

## Which tests are good candidates?

Often it's easier to identify test cases that aren't good candidates:

 * Negative test
 * Single unit test function that cover only one little aspect (like listing volumes)
 * Tests that interfere each other (like changing quotas)
 
 All others are might good candidates to integrate them. So please feel free to identify new cases and to upload patches in OpenStack/Tempest.
