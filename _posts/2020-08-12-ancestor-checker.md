---
layout: post
title:  "Validating 2nd Gen Package Ancestory"
date:   2020-08-12
categories: SFDX packaging
author: john
---

We love all of the new development tools Salesforce is building lately and we especially love 2nd generation packaging. One neat feature of 2nd generation packages is [flexible branch versioning](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_dev2gp_config_ancestors.htm). With this feature you can abandon package versions that you no longer want to build on! This allows you to effectively roll back changes to global method signatures or the data model of your application. 

The caveat here is that there still should be a single upgrade path for customers (as there is for 1st generation packages). That is, there should be an unbroken chain from the earliest working version to the latest working version, without branches. If there are forks, or multiple roots, then your customer may install the wrong package version and reach a dead end - unable to upgrade to the latest and greatest! In the worst case you might end up maintaining multiple version trees for each of your customers. Although Salesforce would probably help you out if things got that dire.

So, you may accidentially set the wrong ancestor version of a package (or forget to set one at all!). If you fail to notice this mistake and, say, install that package version into a customer org then you'd be in quite the pickle! Now your customers are on different branches and, barring Salesforce intervention or an expensive uninstall/reinstall scenario, they will stay on separate branches! This is what we call a 'bad fork'.

A bad fork is when two releases point to the same ancestor. For example:

    base - v1.0 - RELEASED
      |---base - 05i4Q000000CaXXXXX - v1.1 
      |---base - 05i4Q000000CaXXXXX - v2.0 

With this bad fork a customer on v1.1 wouldn't be able to upgrade to v2.0.

We've written a small script to identify bad forks and we run an hourly Jenkins job to catch them. Eventually we'd like to add this as a SFDX CLI plugin.

The script runs `sfdx force:package:version:list --released` & throws exception if there is a bad fork in the released packages

If there is a bad fork then rename the released package using the below command to 'DO NOT USE'. 

`sfdx force:package:version:update -p 05i4Q000000CaXXXXX -a "DO NOT USE"`

Here's an example!

Sample output

    > node hierarchy_validate.js me@dev.hub
    |--base - 05i4Q000000CaXXXXX - v1.0 - 1.0.0.3 - RELEASED
      |--base - 05i4Q000000CaXXXXX - v1.1 - 1.1.0.1 - RELEASED
    |--bread - 05i4Q000000CaXXXXX - v1.0 - 0.2.0.1 - RELEASED
      |--bread - 05i4Q000000CaXXXXX - v1.1 - 1.1.0.3 - RELEASED
    |--butter - 05i4Q000000CaXXXXX - v1.0 - 1.0.0.7 - RELEASED
      |--butter - 05i4Q000000CaXXXXX - v1.1 - 1.1.0.4 - RELEASED
        |--butter - 05i4Q000000CaXXXXX - v1.2 - 1.2.0.1 - RELEASED
    |--jam - 05i4Q000000CaXXXXX - v0.1 - 0.1.0.4 - RELEASED
      |--jam - 05i4Q000000CaXXXXX - v0.3.0 - 0.3.0.1 - RELEASED
        |--jam - 05i4Q000000CaXXXXX - v1.3 - 1.3.0.6 - RELEASED
          |--jam - 05i4Q000000CaXXXXX - v1.3.1 PATCH 1 - 1.3.1.1 - RELEASED
          |--jam - 05i4Q000000CaXXXXX - v1.4 - 1.4.0.3 - RELEASED
    validating package: base
    base is good!
    validating package: bread
    bread is good!
    validating package: butter
    butter is good!
    validating package: jam
    jam is good!

Here's the script!

    /*
    USAGE EXAMPLE: node hierarchy_validate.js [username]

    Runs `sfdx force:package:version:list --released -v [username]` & throws exception if there is a bad fork in the released packages
    */
    const { assert } = require('console');

    const IGNORE_TITLE = 'DO NOT USE';
    let LOGGING_ENABLED = false;
    class Version {
      constructor(major, minor, patch, build) {
        this.major = major;
        this.minor = minor;
        this.patch = patch;
        this.build = build;
      }

      is_patch_of(targetVersion) {
        return targetVersion.major == this.major && targetVersion.minor == this.minor;
      }

      /**
      * @param targetVersion
      *
      * @return true if this version is prior to the target version
      */
      is_prior_to(targetVersion) {
        if (targetVersion.major > this.major) {
          return true;
        } else if (targetVersion.major == this.major) {
          if (targetVersion.minor > this.minor) {
            return true;
          } else if (targetVersion.minor == this.minor) {
            if (targetVersion.patch > this.patch) {
              return true;
            } else if (targetVersion.patch == this.patch) {
              if (targetVersion.build > this.build) {
                return true;
              }
            }
          }
        }
        return false;
      }
    }

    let build_package_version = (package) => {
      return new Version(package.MajorVersion, package.MinorVersion, package.PatchVersion, package.BuildNumber);
    };

    // appends errors in hierarchy to pkgErrors
    let validate_hierarchy = (
      roots,
      pkgErrors,
      currentVersionNumber = new Version(0, 0, 0, 0) // start at pkgroot
    ) => {
      let nextNonPatchRelease;
      // validate that there is only a single child with a greater version number (can ignore patches)
      for (pkgroot of roots) {
        let version = build_package_version(pkgroot);
        if (version.is_patch_of(currentVersionNumber)) {
          // Can have as many released patches of a version that we want
          continue;
        } else if (!currentVersionNumber.is_prior_to(version)) {
          // something wrong with the script
          pkgErrors.push(`child ${pkgroot.Id}, ${version} is earlier than parent version ${currentVersionNumber}??`);
        } else if (nextNonPatchRelease) {
          pkgErrors.push(
            `BAD FORK! Found two promoted releases pointing to same ancestor: ${pkgroot.Id}, ${nextNonPatchRelease.Id}`
          );
        } else {
          nextNonPatchRelease = pkgroot;
        }
      }
      if (nextNonPatchRelease) {
        return validate_hierarchy(nextNonPatchRelease.children, pkgErrors, build_package_version(nextNonPatchRelease));
      }
    };

    let serialize_to_line = (package) => {
      let released = package.IsReleased ? 'RELEASED' : 'notreleased';
      return package.Package2Name + ' - ' + package.Id + ' - ' + package.Name + ' - ' + package.Version + ' - ' + released;
    };

    let print_hierarchy = (roots, depth = 0) => {
      let indent = '  '.repeat(depth);
      for (const package of roots) {
        if (LOGGING_ENABLED) {
          console.log(indent + '|--' + serialize_to_line(package));
        }
        print_hierarchy(package.children, depth + 1);
      }
    };

    let get_roots = (managedPackages, raw) => {
      const result = JSON.parse(raw);

      packageNames = new Set();
      pvById = {};
      roots = [];
      if (result.status != 0) {
        throw new Error('error executing sfdx package version command');
      } else {
        for (const package of result.result) {
          package.children = [];
          if (!managedPackages.includes(package.Package2Name)) {
            continue;
          }
          if (!package.IsReleased) {
            continue;
          }
          if (package.Name == IGNORE_TITLE) {
            continue;
          }
          packageNames.add(package.Package2Name);
          pvById[package.SubscriberPackageVersionId] = package;
        }

        for (const package of result.result) {
          if (pvById[package.SubscriberPackageVersionId]) {
            if (package.AncestorId != 'N/A' && package.AncestorId != null) {
              if (!pvById[package.AncestorId]) {
                throw new Error(
                  `package ${package.SubscriberPackageVersionId} has a bad fork ancestor: ${package.AncestorId}`
                );
              }
              pvById[package.AncestorId].children.push(package);
            } else {
              roots.push(package);
            }
          }
        }
      }

      return {
        packageNames: Array.from(packageNames),
        roots: roots
      };
    };

    let get_managed_packages = (raw) => {
      const result = JSON.parse(raw);

      if (result.status != 0) {
        throw new Error('error executing sfdx package list command');
      } else {
        let managedPackages = [];
        for (const package of result.result) {
          if (package.ContainerOptions == 'Managed') {
            managedPackages.push(package.Name);
          }
        }
        return managedPackages;
      }
    };

    let run = (packageListData, packageVersionData) => {
      managedPackages = get_managed_packages(packageListData);
      const { packageNames, roots } = get_roots(managedPackages, packageVersionData);
      print_hierarchy(roots);
      // for each package
      let errors = [];
      for (const packageName of packageNames) {
        if (LOGGING_ENABLED) {
          console.log(`validating package: ${packageName}`);
        }
        let pkgErrors = [];
        validate_hierarchy(
          // filter out roots not related to package name
          roots.filter((pkgroot) => pkgroot.Package2Name == packageName),
          pkgErrors
        );
        if (pkgErrors.length == 0) {
          if (LOGGING_ENABLED) {
            console.log(`${packageName} is good!`);
          }
        } else {
          if (LOGGING_ENABLED) {
            console.log(`found error(s) in package ${packageName}: ${pkgErrors}`);
          }
          errors = errors.concat(pkgErrors);
        }
      }
      if (errors.length != 0) {
        throw new Error(errors);
      }
    };

    let run_tests = (successtests, failtests) => {
      const { packages } = require('./test/hierarchy_validate_all_packages');
      for (test of successtests) {
        run(packages, test);
      }
      for (test of failtests) {
        try {
          run(packages, test);
          assert(false, 'should have failed');
        } catch (ex) {
          assert(ex.message.includes('BAD FORK') || ex.message.includes('bad fork'));
        }
      }
    };

    // for testing
    let run_from_file = (filename) => {
      const fs = require('fs');
      fs.readFile(filename, 'utf8', (err, data) => {
        run(data);
      });
    };

    let run_from_command = () => {
      let args = process.argv.slice(2);
      let usernameArg = args.length > 0 ? '-v ' + args[0] : '';
      const { exec } = require('child_process');
      exec(`sfdx force:package:list --json ${usernameArg}`, (error, stdout, stderr) => {
        if (error) {
          console.log(`error: ${error.message}`);
          exit(1);
        }
        if (stderr) {
          console.log(`stderr: ${stderr}`);
          exit(1);
        }
        packageList = stdout;
        exec(`sfdx force:package:version:list --released --json ${usernameArg}`, (error, stdout, stderr) => {
          if (error) {
            console.log(`error: ${error.message}`);
            exit(1);
          }
          if (stderr) {
            console.log(`stderr: ${stderr}`);
            exit(1);
          }
          run(packageList, stdout);
        });
      });
    };

    const { exit } = require('process');

    LOGGING_ENABLED = true;
    // run_from_file('all_packages.json', 'all_package_versions.json');
    run_from_command();