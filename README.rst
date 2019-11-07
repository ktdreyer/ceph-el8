Building Ceph's dependencies for EPEL 8
=======================================

Our strategy is to fork packages from Fedora Rawhide into EPEL 8.

As an intermediate step, we will build these in a Copr first.

Clone the source code for a Fedora package
------------------------------------------

Ensure your SSH public key is set up in `FAS
<https://admin.fedoraproject.org/accounts/>`_. Install ``fedpkg``, then clone
a package (eg. python-remoto) like so::

    fedpkg clone python-remoto

Locally build a Fedora package using mock
-----------------------------------------

In the dist-git clone directory::

    fedpkg mockbuild

fedpkg will build the package in a local mock chroot. This chroot will be the
version of Fedora that corresponds to the dist-git branch you have checked
out. If you have checked out the ``master`` branch, this will build for Rawhide.
If you have checked out the ``f30`` branch, then this will build in a Fedora
30 buildroot. If you have checked out the ``epel8`` branch, fedpkg will build
with an EPEL 8 buildroot.

Override the buildroot without branching
----------------------------------------

fedpkg looks at your current branch in order to determine which mock
configuration (ie buildroot) to use when building. You can override this with
the ``--release`` flag. In the dist-git clone directory::

    git checkout master
    fedpkg --release epel8 mockbuild

This allows you to quickly test building the "master" code (Rawhide) for EPEL
8.

Analyzing EPEL 8 dependency failures
------------------------------------

Here's one package that is missing several dependencies on EPEL 8:
python-cherrypy. Let's try building it::

    fedpkg clone python-cherrypy
    cd python-cherrypy
    fedpkg --release epel8 mockbuild

    [... lots of build output snipped ... ]

    No matching package to install: 'python3-zc-lockfile'
    No matching package to install: 'python3dist(cheroot)'
    No matching package to install: 'python3dist(nose-testconfig)'
    No matching package to install: 'python3dist(path.py)'
    No matching package to install: 'python3dist(portend)'

    Not all dependencies satisfied
    Error: Some packages could not be found.

This tells us the **build-time** dependencies for python-cherrypy. We need to
build these for EPEL 8 *first* and then we can build python-cherrypy.

Also note here: It is worth looking closely at this list to make sure that
they are all *really* necessary. For example Alfredo found that
nose-testconfig is not really required, and it was simply a bug in the
packaging that we will `fix
<https://src.fedoraproject.org/rpms/python-cherrypy/pull-request/5>`_ in
Rawhide (master).

Intro to Copr
-------------

Fedora's `Copr <https://fedoraproject.org/wiki/Category:Copr>`_ is like
`Ubuntu's PPAs <https://launchpad.net/ubuntu/+ppas>`_: it's a way to build and
host one or more packages for a distribution. It has a web interface and a
CLI. We will use an "`ceph-el8
<https://copr.fedorainfracloud.org/coprs/ktdreyer/ceph-el8/>`_" Copr for this
project.

Building a Rawhide SRPM in Copr
-------------------------------

Let's start with building one of the easy cherrypy dependencies above:
`python-path <https://src.fedoraproject.org/rpms/python-path>`_::

    fedpkg clone python-path
    cd python-path

First, generate an SRPM from the current branch (eg. "master" for Rawhide)::

    fedpkg srpm
    Wrote: /home/kdreyer/fedora-scm/python-path/python-path-11.5.0-2.fc32.src.rpm

There are two ways to submit this resulting SRPM file to Copr: the web
interface, or the CLI.

Here is how to do the build with the web UI:

1. Log into the `Copr web UI <>`_
2. Click the "Builds" tab
3. Under "Select the source type", click "Upload"
4. Under "Provide the source", click "Choose File"
5. Upload your ``python-path-11.5.0-2.fc32.src.rpm`` file from your laptop
6. Click the blue "Build" button

Here is how to do the build with the CLI:

1. Log into the Copr web UI
2. Get your API token at the `Copr API Page
   <https://copr.fedorainfracloud.org/api/>`_
3. Save the token to your system::

    vim ~/.config/copr

4. Test that your authentication works::

    copr-cli whoami
    ktdreyer

5. Run the ``build`` command with your SRPM::

    copr-cli build ktdreyer/ceph-el8 python-path-11.5.0-2.fc32.src.rpm

Monitor the ``builds`` web page to see the newly-built package.

Using Copr repo with mock
-------------------------

Now we have two repositories that we want to use when we build cherrypy
locally with mock: **epel8** plus the **el8-ceph** Copr.

In order to use both repositories with mock, we need a custom mock
configuration file. You can find that file in this Git repository. Here is how
to install (symlink) it into place (``/etc/mock``)::

    sudo -i
    cd /etc/mock
    ln -s /home/kdreyer/path/to/ceph-el8/el8-ceph-x86_64.cfg

Let's try building the cherrypy package again, using our custom mock config
that points at the Copr::

    cd python-cherrypy
    fedpkg --release epel8 mockbuild --root el8-ceph-x86_64

This time the build fails with the list of ``No matching package``, but you
should not see ``python-path`` in that list any more.


Conclusion
----------

We will iterate through the list of missing epel8 packages until we can
completely build cherrypy. Once that is done, we will branch all the packages
that we need in dist-git. Then we will do the real builds in Fedora's koji and
push those as updates to epel8 in Bodhi.
