Releasing Titan
===============

Prerequisites
-------------

The release process has only been tested on Linux.  The following
tools must be installed.

* [expect](http://expect.sourceforge.net/)
* [gollum-site](https://github.com/dreverri/gollum-site)
* [gpg](http://www.gnupg.org/) with a running agent

~/.m2/settings.xml will need the following entries.

```xml
<settings>
  <servers>
    <server>
      <id>sonatype-nexus-snapshots</id>
      <username>...</username>
      <password>...</password>
    </server>
    <server>
      <id>sonatype-nexus-staging</id>
      <username>...</username>
      <password>...</password>
    </server>
    <server>
      <id>aurelius.s3</id>
      <username>AWS Key ID</username>
      <password>AWS Secret Key</password>
    </server>
  </servers>
</settings>
```

Release Steps
-------------


### Updating Titan's titan.compatible.versions pom.xml property

The property titan.compatible.versions is a comma-separated string 
listing the Titan versions which create databases in a storage format
compatible with the current runtime version.
Titan will only open databases whose version string is either the
the live Titan runtime version or one of the elements in this list.

For releases that break storage format compatibility, this property
should be emptied.  For releases  that maintain backwards compatibility,
this list should be updated to include the most recent compatible
version(s).

The property should look something like this near the top of pom.xml:

```bash
    <titan.compatible.versions>0.4.0,0.4.1,0.4.2</titan.compatible.versions>
```

### Preparing Documentation

Update version-sensitive wiki pages and doc/HTML files in the repo.

Wiki pages:

* Home
* Release-Notes
* Version-Compatibility
* Upgrade-Instructions
* Acknowledgements
* Downloads

Files in the main repo:

* CHANGELOG.textile
* NOTICE.txt
* UPGRADE.textile
* titan-site/src/site-resources/index.html
  (this template generates a new http://thinkaurelius.github.io/titan/)

Some of these updates could potentially be automated, but tweaking
documentation before a release will always require at least human in
the loop.

### Preparing the Local Repository

Commit or stash any uncommitted changes.

Recommended but not required:

* Push any unpushed commits on master with <code>git push origin
  master</code>
* Delete untracked files and directories in the clone by running
  <code>git clean -fd</code>

### Deploying Artifacts & Archives

```bash
# This does not push anything.  It creates two local commits and a tag.
# You will be prompted for the release version and the next dev version.
mvn release:prepare -Prelease-plugin-hack

# The tag name generated by release:prepare does not match the general
# pattern established before we started using the release plugin.  This
# script deletes the tag and creates it with a pattern-conformant name.
titan-dist/target/util-scripts/rewrite-tag.sh release.properties

# For the rest of this document, $RELEASE_VERSION is the version we're
# in the process of releasing, e.g. 0.4.1
RELEASE_VERSION=`sed -nr 's/^scm\.tag=//p' release.properties`
echo RELEASE_VERSION: $RELEASE_VERSION

# Proceeding beyond this comment commits you to either complete the
# rest of the release steps or, if you must abort, to cleanup
# afterward.  The next command is world-visible in that it deploys
# jars to Sonatype OSS staging and uploads .zip/.tar.bz2 archives to
# S3.  It's not a huge deal -- the Sonatype staging artifacts won't
# show up on Maven Central without additional steps and there will be
# no links pointing people to the artifacts in S3 -- but it's still
# technically world-visible.  In contrast, commands above this comment
# were limited to local effects invisible to the wider Internet.
git checkout refs/tags/$RELEASE_VERSION
mvn clean javadoc:jar deploy -Paurelius-release
# You can append -DskipTests=true above if you've already run the tests
#
# Delete titan-dist folder from Sonatype OSS if it exists
```

### Deploying Artifacts & Archives for JRE6

Titan can't be compiled on Oracle JDK6 due to a limitation/bug in the
Java 6 compiler.  However, Titan maintains source-level compatibility
with Java 6.  JDK7 can cross-compile Titan to the JRE6 runtime target.
To do this, the JDK7 compiler needs a copy of rt.jar from Java 6
configured in the so-called boot classpath.

To build artifacts for JRE6 and deploy them to S3 and Sonatype OSS
staging:

* Install Oracle JRE6 or JDK6 if you don't already have it installed

* Find the path to the file named rt.jar inside your JRE6/JDK6 install
  and note it for future use (e.g. `/opt/jdk1.6.0_45/jre/lib/rt.jar`)

```bash
# Checkout the release tag if not already checked out.
# $RELEASE_VERSION should be set as described in the previous section.
git checkout refs/tags/$RELEASE_VERSION

# Rewrite every pom.xml file in the project using jre6.xslt.  This
# changes the maven-compiler-plugin configuration to target Java 6.
# More importantly, it also suffixes the artifactIds of all titan
# artifacts with "-jre6".  This name mangling ensures that the JRE6
# artifacts occupy a dependency tree separate from the JRE7 artifacts,
# so that the artifacts can be safely pushed to central without
# risking mixed-runtime dependency trees.  It also detaches the
# Persistit module from the build cycle.
mvn clean xml:transform resources:copy-resources -Ptarget-jre6,release-plugin-hack

# Compile, package, and deploy artifacts for JRE6.  Do not use JDK6
# javac here.  Use JDK7 javac but include the Xbootclasspath argument
# pointing to rt.jar from JDK6/JRE6.
mvn clean javadoc:jar deploy -Paurelius-release -Dcompiler.args=-Xbootclasspath:/path/to/java6/lib/rt.jar
```

### Checking Artifacts & Archives

This is a good time to inspect the archives just uploaded to
http://s3.thinkaurelius.com/downloads/titan/.  Directory listing is
normally disabled on that directory, so you may need to login to the
S3 console to list the contents and check that nothing's out of place.

If S3 looks good, then inspect and close the staging repository that
the deploy phase automatically created on https://oss.sonatype.org/.

If you decide to abort at this stage, then:

* Delete any artifacts just uploaded to S3
* Drop the staging repository in Sonatype OSS
* Run the following commands to delete your local git changes

```bash
#
#                        WARNING
#
# This will obliterate all local commits and uncommitted files.
# Only copy and paste this code segment if you want to abort a
# partially-completed release.  Replace $RELEASE_VERSION by the
# version just attempted.
git checkout master
git reset --hard origin/master
git tag -d $RELEASE_VERSION
git clean -fd
```

Assuming everything looked fine on OSS Sonatype and S3, we'll move on
to updating gh-pages locally.

### Generating new Javadoc & Wikidoc for gh-pages

```bash

# This clones the repo to /tmp/titanpages.  It checks out the
# gh-pages branch, then populates wikidoc/$RELEASE_VERSION/ and
# javadoc/$RELEASE_VERSION/ with a copy of the wiki exported using
# gollum-site and a copy of the current javadoc (respectively).  It
# also replaces the wikidoc/current/ and javadoc/current/ directories
# with copies of files for $RELEASE_VERSION.  Finally, it generates a
# new index.html.  It commits these changes and pushes them to the
# local parent repo (but not github).
#
# Why clone to /tmp/titanpages?  We need to modify the gh-pages branch,
# but we don't want to risk file conflicts between the doc subfolder
# of the gh-pages branch and the doc submodule of the titan master
# branch.  Cloning the repo before checking out gh-pages guarantees a
# clean working copy and thereby precludes conflicts on doc (or anything
# else that might become conflict-prone in the future).
titan-site/target/site-scripts/gh-pages-update.sh release.properties

# If you want to check this script's work, then consider running
# the following:
git diff --name-status origin/gh-pages..gh-pages
git diff origin/gh-pages..gh-pages | less # lots of text
```

### Release to Maven Central and Push to Github

Release the Sonatype OSS repository that you inspected and closed
earlier.  It will appear on Maven Central in an hour or two.

Finally, push your local changes to Github:

```bash 
# cd to the titan repository root if not already there
git push origin master
git push origin refs/tags/$RELEASE_VERSION
git push origin gh-pages
```

### Deploy a New Snapshot

To kickoff the next round of development, deploy a copy of the new
SNAPSHOT version's artifacts to the Sonatype OSS snapshots repository.

```bash
# From the repository root
git checkout master
mvn clean deploy
```
