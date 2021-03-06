# Copyright (C) 2013-2016 DNAnexus, Inc.
#
# This file is part of dx-toolkit (DNAnexus platform client libraries).
#
#   Licensed under the Apache License, Version 2.0 (the "License"); you may not
#   use this file except in compliance with the License. You may obtain a copy
#   of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
#   WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
#   License for the specific language governing permissions and limitations
#   under the License.

SHELL=/bin/bash -e

ifndef MAKEOPTS
	ifneq ($(OS), Windows_NT)
		MAKEOPTS=-j -l 2.0
	endif
endif
MAKE:=$(MAKE) $(MAKEOPTS)

# Set the minimum Windows version (required by the ua and libcurl.la targets).
ifndef CPPFLAGS
	ifeq ($(OS), Windows_NT)
		export CPPFLAGS=-DWINVER=0x0501
	endif
endif

UNAME := $(shell uname)

CENTOS_MAJOR_VERSION := $(shell test -e /etc/issue && (grep -o "CentOS release [0-9]\+" /etc/issue | sed -e "s/CentOS release //"))
FEDORA_MAJOR_VERSION := $(shell test -e /etc/issue && (grep -o "Fedora release [0-9]\+" /etc/issue | sed -e "s/Fedora release //"))
UBUNTU_VERSION := $(shell test -e /etc/issue && (grep -o "Ubuntu [0-9]\+\.[0-9]\+" /etc/issue | sed -e "s/Ubuntu //"))
RHEL_MAJOR_VERSION := $(shell test -e /etc/redhat-release && (grep -o "Red Hat Enterprise Linux .* release [0-9]\+" /etc/redhat-release | sed -e "s/Red Hat Enterprise Linux .* release //"))

ifndef CCACHE_DISABLE
	export PATH := /usr/lib/ccache:$(PATH)
endif

# If installing into the system directories you probably want to set
#   DESTDIR=/ PREFIX=/usr
ifndef DESTDIR
	export DESTDIR=/opt/dnanexus
endif
ifndef PREFIX
	export PREFIX=/
endif

export DNANEXUS_HOME=$(CURDIR)/..
export PATH := $(DNANEXUS_HOME)/build/bin:$(PATH)
export DX_PY_ENV=$(DNANEXUS_HOME)/build/py_env
export DNANEXUS_LIBDIR := $(DNANEXUS_HOME)/share/dnanexus/lib

# Short-circuit sudo when running as root. In a chrooted environment we are
# likely to be running as root already, and sudo may not be present on minimal
# installations.
ifeq (${USER}, root)
	MAYBE_SUDO=
else
	MAYBE_SUDO='sudo'
endif

all: api_wrappers cpp src_libs python dx-verify-file jq dx-docker

# Don't try to initialize the submodule in source distributions, it should
# already be there and ready to build.
git_submodules:
	(if [ -d "$(DNANEXUS_HOME)/.git" ]; then \
	    cd ..; \
	    git submodule init; \
	    "`which git-retry || which git`" submodule update; \
	fi)

# Client bindings and wrappers
# ============================

toolkit_version:
	(if [ ! -e "$(DNANEXUS_HOME)/src/python/dxpy/toolkit_version.py" ]; then \
	    echo "version = 'UNKNOWN_VERSION'" > "$(DNANEXUS_HOME)/src/python/dxpy/toolkit_version.py"; \
	    echo -e "package com.dnanexus;\nclass DXToolkitVersion {\n    public static final String TOOLKIT_VERSION = \"0.0.0\";\n}" > "$(DNANEXUS_HOME)/src/java/src/main/java/com/dnanexus/DXToolkitVersion.java"; \
	    echo UNKNOWN_VERSION > "$(DNANEXUS_HOME)/build/info/version"; \
	fi)
	(if [ -d "$(DNANEXUS_HOME)/.git" ]; then \
	    export GIT_TOOLKIT_VERSION=$$((git describe --match 'v*.*.*' || git rev-parse --short HEAD) | sed -e 's/-[0-9]-/+/'); \
	    echo "version = '$${GIT_TOOLKIT_VERSION/#v/}'" > "$(DNANEXUS_HOME)/src/python/dxpy/toolkit_version.py"; \
	    echo -e "package com.dnanexus;\nclass DXToolkitVersion {\n    public static final String TOOLKIT_VERSION = \"$$GIT_TOOLKIT_VERSION\";\n}" > "$(DNANEXUS_HOME)/src/java/src/main/java/com/dnanexus/DXToolkitVersion.java"; \
	    echo $$GIT_TOOLKIT_VERSION > "$(DNANEXUS_HOME)/build/info/version"; \
	fi)

python/dxpy/api.py: api_wrappers/wrapper_table.json api_wrappers/generatePythonAPIWrappers.py
	cat api_wrappers/wrapper_table.json | api_wrappers/generatePythonAPIWrappers.py > python/dxpy/api.py

cpp/dxcpp/api.h: api_wrappers/wrapper_table.json api_wrappers/generateCppAPIHWrappers.py
	cat api_wrappers/wrapper_table.json | api_wrappers/generateCppAPIHWrappers.py > cpp/dxcpp/api.h

cpp/dxcpp/api.cc: api_wrappers/wrapper_table.json api_wrappers/generateCppAPICCWrappers.py
	cat api_wrappers/wrapper_table.json | api_wrappers/generateCppAPICCWrappers.py > cpp/dxcpp/api.cc

java/src/main/java/com/dnanexus/DXAPI.java: api_wrappers/wrapper_table.json api_wrappers/generateJavaAPIWrappers.py
	cat api_wrappers/wrapper_table.json | api_wrappers/generateJavaAPIWrappers.py > java/src/main/java/com/dnanexus/DXAPI.java

R/dxR/R/api.R: api_wrappers/wrapper_table.json api_wrappers/generateRAPIWrappers.py
	cat api_wrappers/wrapper_table.json | api_wrappers/generateRAPIWrappers.py > R/dxR/R/api.R

api_wrappers: toolkit_version python/dxpy/api.py cpp/dxcpp/api.h cpp/dxcpp/api.cc java/src/main/java/com/dnanexus/DXAPI.java R/dxR/R/api.R
	$(MAKE) -C ../contrib api_wrappers

cpp: api_wrappers
	mkdir -p "$(DNANEXUS_HOME)/share/dnanexus/src"
	cp -a cpp "$(DNANEXUS_HOME)/share/dnanexus/src"

src_libs: api_wrappers
	mkdir -p "$(DNANEXUS_LIBDIR)"
	cp -R -v bash "$(DNANEXUS_LIBDIR)"

PYTHON_LIBDIR := $(DNANEXUS_LIBDIR)/python2.7/site-packages

python: api_wrappers toolkit_version $(DNANEXUS_HOME)/bin/dx

ifeq ($(OS), Windows_NT)
        ACTIVATE=Scripts/activate
	# On Windows using DNANEXUS_HOME as the pip install --prefix results in
	# an oddly nested dir, so make a new var to hold the staging dir name in
	# which we'll be installing packages:
	WHEEL_TEMPDIR=wheel_tempdir
else
        ACTIVATE=bin/activate
endif

# Get the short toolkit version string, resembling v0.195.0
GIT_TOOLKIT_VERSION_SHORT := $(shell git describe --abbrev=0)
# IF upload agent is built, dlls will be placed here, can be overriden with argument
# e.g. `make DLL_DEPS_FOLDER=C:\\toolkit-deps\\ pynsist_installer`
DLL_DEPS_FOLDER = '../../src/ua/dist/'

pynsist_installer: toolkit_version $(DNANEXUS_HOME)/bin/dx dx-verify-file jq

	# Create installer.cfg and insert the toolkit version string
	export GIT_TOOLKIT_VERSION_SHORT=$(GIT_TOOLKIT_VERSION_SHORT) ; \
	    sed s/TEMPLATE_STRING_TOOLKIT_VERSION/$${GIT_TOOLKIT_VERSION_SHORT}/ "$(DNANEXUS_HOME)"/build/pynsist_files/installer.cfg.template > "$(DNANEXUS_HOME)"/build/pynsist_files/installer.cfg
	# Insert dll foldername
	export DLL_DEPS_FOLDER=$(DLL_DEPS_FOLDER) ; \
		sed --in-place "s~DLL_DEPS_FOLDER~$${DLL_DEPS_FOLDER}~" "$(DNANEXUS_HOME)"/build/pynsist_files/installer.cfg

	# Copy wheel file into place without changing its filename
	export DXPY_WHEEL_FILENAME=$$(basename $(DNANEXUS_HOME)/src/python/dist/dxpy-*.whl) ; \
	    cp "$(DNANEXUS_HOME)"/src/python/dist/$${DXPY_WHEEL_FILENAME} "$(DNANEXUS_HOME)"/build/pynsist_files/

	# Download the .whl files for each of dxpy's Python dependencies, to bundle with NSIS installer
	unset PYTHONPATH; source "$(DX_PY_ENV)/$(ACTIVATE)"; mkdir -p "$(DNANEXUS_HOME)"/build/pynsist_files/wheelfile_depends; python -m pip install --download "$(DNANEXUS_HOME)"/build/pynsist_files/wheelfile_depends -r python/requirements.txt -r python/requirements_windows.txt

	# Insert the wheel filename into installer.cfg
	export DXPY_WHEEL_FILENAME=$$(basename $(DNANEXUS_HOME)/src/python/dist/dxpy-*.whl) ; \
	    sed --in-place s/TEMPLATE_STRING_DXPY_WHEEL_FILENAME/$${DXPY_WHEEL_FILENAME}/ "$(DNANEXUS_HOME)"/build/pynsist_files/installer.cfg

	# Create pyapp_dnanexus.nsi and insert the wheel filename into it
	export DXPY_WHEEL_FILENAME=$$(basename $(DNANEXUS_HOME)/src/python/dist/dxpy-*.whl) ; \
	    sed s/TEMPLATE_STRING_DXPY_WHEEL_FILENAME/$${DXPY_WHEEL_FILENAME}/ "$(DNANEXUS_HOME)"/build/pynsist_files/pyapp_dnanexus.nsi.template > "$(DNANEXUS_HOME)"/build/pynsist_files/pyapp_dnanexus.nsi

	# Install the pynsist package and use it to build a Windows installer
	unset PYTHONPATH; source "$(DX_PY_ENV)/$(ACTIVATE)"; python -m pip install pynsist==1.12
	unset PYTHONPATH; source "$(DX_PY_ENV)/$(ACTIVATE)"; pynsist "$(DNANEXUS_HOME)"/build/pynsist_files/installer.cfg
	# Copy .exe installer to dx-toolkit root once it's ready
	cp "$(DNANEXUS_HOME)"/build/pynsist_files/build/nsis/DNAnexus_CLI_"$(GIT_TOOLKIT_VERSION_SHORT)".exe "$(DNANEXUS_HOME)"/dx-toolkit-"$(GIT_TOOLKIT_VERSION_SHORT)".exe

$(DNANEXUS_HOME)/bin/dx: $(shell find python/{dxpy,scripts,requirements*,setup*} -not -name toolkit_version*)
	python -c 'import sys; exit("dx-toolkit is not compatible with Python < 2.7" if sys.version_info < (2, 7) else 0)'
	rm -rf "$(PYTHON_LIBDIR)" "$(DX_PY_ENV)" python/dist
	mkdir -p "$$(dirname '$(PYTHON_LIBDIR)')"
	virtualenv --python=python "$(DX_PY_ENV)"

	# Install setuptools and other fundamental packages
ifeq ($(OS), Windows_NT)
	# Install setuptools and build the dxpy wheel
	unset PYTHONPATH; source "$(DX_PY_ENV)/$(ACTIVATE)"; python -m pip install --upgrade -r python/requirements_setuptools.txt
else
	unset PYTHONPATH; source "$(DX_PY_ENV)/$(ACTIVATE)"; pip install --upgrade -r python/requirements_setuptools.txt
endif

	# On OS X, install the gnureadline package and download other osx specific packages
ifeq ($(UNAME), Darwin)
	unset PYTHONPATH; source "$(DX_PY_ENV)/$(ACTIVATE)"; pip install --no-cache-dir --prefix="$(DNANEXUS_HOME)" -r python/requirements_readline.txt
endif

	# Build the dxpy wheel and move it into place
	unset PYTHONPATH; source "$(DX_PY_ENV)/$(ACTIVATE)"; cd python; python setup.py bdist_wheel
ifeq ($(OS), Windows_NT)
	# Install the dxpy .whl and wheel itself to the temp dir. Note wheel is only
	# needed because we need to bundle it with the pynsist installer.
	unset PYTHONPATH; source "$(DX_PY_ENV)/$(ACTIVATE)"; python -m pip install --ignore-installed --prefix="$(WHEEL_TEMPDIR)" python/dist/*.whl
	unset PYTHONPATH; source "$(DX_PY_ENV)/$(ACTIVATE)"; python -m pip install --ignore-installed --prefix="$(WHEEL_TEMPDIR)" wheel
	# Move the staging dir artifacts into their final homes
	mv "$(DNANEXUS_HOME)"/src/$(WHEEL_TEMPDIR)/Lib/site-packages "$(PYTHON_LIBDIR)"
	mv "$(DNANEXUS_HOME)"/src/$(WHEEL_TEMPDIR)/Scripts/* "$(DNANEXUS_HOME)"/bin
else
	unset PYTHONPATH; source "$(DX_PY_ENV)/$(ACTIVATE)"; pip install --ignore-installed --prefix="$(DNANEXUS_HOME)" python/dist/*.whl
	mv "$(DNANEXUS_HOME)"/lib/python?.?/site-packages "$(PYTHON_LIBDIR)" || mv "$(DNANEXUS_HOME)"/site-packages "$(PYTHON_LIBDIR)"
endif
	rm -f "$(DNANEXUS_HOME)/bin/xattr"

# If running in fakeroot, override some maven settings so that maven can still
# find a localRepository to write to.
java: api_wrappers toolkit_version
	mkdir -p "$(DNANEXUS_HOME)/lib/java"
	(cd java; if [ "$(FAKED_MODE)" == "" ]; then mvn compile assembly:single; else mvn -s m2-settings.xml compile assembly:single; fi)
	cp -f java/target/*.jar "$(DNANEXUS_HOME)/lib/java"

R: api_wrappers toolkit_version
	mkdir -p "$(DNANEXUS_HOME)/lib/R"
	(cd R; export R_LIBS_SITE="$(DNANEXUS_HOME)/lib/R"; R -q --no-restore --no-save < install_dxR_build_deps.R; R -q --no-restore --no-save -e "library(roxygen2); roxygenize(\"dxR\")"; R CMD INSTALL --build dxR --library="$(DNANEXUS_HOME)/lib/R"; )


# Tools
# =====

dx-verify-file: api_wrappers
	$(MAKE) -C dx-verify-file install DESTDIR=$(DNANEXUS_HOME) PREFIX=/

ua: toolkit_version ua_deps
	$(MAKE) -C ua install dist DESTDIR=$(DNANEXUS_HOME) PREFIX=/

dx-docker: bin/proot bin/docker2aci

bin/proot:
	curl -k -sL https://github.com/proot-me/proot-static-build/raw/5bcf1a3341beeefde7305ce196031de837ca173c/static/proot-x86_64 > $(DNANEXUS_HOME)/bin/proot && chmod 777 $(DNANEXUS_HOME)/bin/proot

bin/docker2aci:
ifeq ($(UNAME), Darwin)
	curl -k -sL https://dl.dnanex.us/F/D/40XJf2gqYqYqzYbGzyqB4g9PxGX9ZY9Bqvgv0Pz5/docker2aci-osx > $(DNANEXUS_HOME)/bin/docker2aci && chmod 777 $(DNANEXUS_HOME)/bin/docker2aci
else ifeq ($(OS), Windows_NT)
    echo "WARNING: dx-docker add-to-applet, upload not currently supported on Windows."
else
	curl -k -sL https://dl.dnanex.us/F/D/B9ZFXqk6q9g0Z3FygYq61y82vpKQ2gj191vkP9jz/docker2aci > $(DNANEXUS_HOME)/bin/docker2aci && chmod 777 $(DNANEXUS_HOME)/bin/docker2aci
endif


# Tests
# =====

test: test_python

coverage_python:
	(cd python; export PYTHONPATH="$$PYTHONPATH:$$(echo $$(pwd)/coverage-*.egg)"; \
	 test/coveragereport.py --strip-prefix="$$(readlink -f $$(pwd))/build/lib.linux-x86_64-2.7" --cover dxpy. --cover scripts --exclude dxpy.packages --exclude EGG-INFO.scripts --exclude test. --exclude setuptools --path-alias "$$(readlink -f $$(pwd)/../../share/dnanexus/lib/python2.7/site-packages/dxpy*.egg)"="$$(readlink -f $$(pwd))" --path-alias "$$(readlink -f $$(pwd)/build/lib.linux-x86_64-2.7)"="$$(readlink -f $$(pwd))")

test_python:
	../build/run_python_integration_tests.py

test_java:
	../build/run_java_integration_tests.py

# Installation
# ============

base_install: all
	mkdir -p $(DESTDIR)/$(PREFIX)/bin
	rm -rf $(DESTDIR)/$(PREFIX)/lib/python2.7/site-packages/dxpy*              # Prevent file-overwriting-dir (or the opposite) problem

	cp -a $(DNANEXUS_HOME)/bin/* $(DESTDIR)/$(PREFIX)/bin/                     # binaries

# Many apps have an autodep that says DESTDIR=/ PREFIX=/opt/dnanexus. Work
# around that and install environment in the correct place until all those apps
# have been replaced.
install: base_install
	cp -a $(DNANEXUS_HOME)/{lib,share} $(DESTDIR)/$(PREFIX)                    # libraries
	rm -f $(DESTDIR)/$(PREFIX)/share/dnanexus/src/cpp/*/.gitignore
	cp -a $(DNANEXUS_HOME)/doc $(DESTDIR)/$(PREFIX)/share/dnanexus             # docs
	(if [ "$(DESTDIR)" == "/" ]; then install -v -m0644 $(DNANEXUS_HOME)/environment $(PREFIX); else install -v -m0644 $(DNANEXUS_HOME)/environment $(DESTDIR); fi)

debian_install: base_install
	cp -a $(DNANEXUS_HOME)/share $(DESTDIR)/$(PREFIX)                          # core libraries
	mkdir -p $(DESTDIR)/$(PREFIX)/share/dnanexus/build
	cp -a $(DNANEXUS_HOME)/build/info $(DESTDIR)/$(PREFIX)/share/dnanexus/build/
	rm -f $(DESTDIR)/$(PREFIX)/share/dnanexus/src/cpp/*/.gitignore
	cp -a $(DNANEXUS_HOME)/doc $(DESTDIR)/$(PREFIX)/share/dnanexus             # docs

	../build/fix_shebang_lines.sh $(DESTDIR)/$(PREFIX)/bin --debian-system-install

	mkdir -p $(DESTDIR)/etc/profile.d                                        # Install environment file into etc
	install -v -m0644 $(DNANEXUS_HOME)/build/environment.redirector $(DESTDIR)/etc/profile.d/dnanexus.environment
	install -v -m0644 $(DNANEXUS_HOME)/environment $(DESTDIR)/etc/profile.d/dnanexus.environment.sh
	cp -a $(DNANEXUS_HOME)/share/dnanexus/lib/python2.7/site-packages/* $(DESTDIR)/$(PREFIX)/share/dnanexus/lib/python2.7/site-packages/
	ls $(DNANEXUS_HOME)/share/dnanexus/lib/python2.7/site-packages | grep dxpy > $(DESTDIR)/$(PREFIX)/share/dnanexus/lib/python2.7/site-packages/dxpy.pth

debian_java_install: java
	mkdir -p $(DESTDIR)/$(PREFIX)/share/java
	$(eval CLEANED_JAR_NAME := `cd "$(DNANEXUS_HOME)"/lib/java; ls *.jar | sed "s/dnanexus-api-\([0-9]\+\.[0-9]\+\.[0-9]\+\)-.*/dnanexus-api-\1.jar/g"`)
	(cd "$(DNANEXUS_HOME)"/lib/java; cp -a *.jar $(DESTDIR)/$(PREFIX)/share/java/"$(CLEANED_JAR_NAME)")

debian_r_install: R
	mkdir -p $(DESTDIR)/$(PREFIX)/lib/R/site-library
	cp -a $(DNANEXUS_HOME)/lib/R/{RCurl,RJSONIO} $(DESTDIR)/$(PREFIX)/lib/R/site-library/
	cp -a $(DNANEXUS_HOME)/lib/R/dxR $(DESTDIR)/$(PREFIX)/lib/R/site-library/

# Documentation
# =============

doc: doc_python doc_dxcpp doc_dxjson doc_java

setup_doc_virtualenv: .setup_doc_virtualenv

.setup_doc_virtualenv: ../build/doc_build_requirements.txt
	@if ! which pip > /dev/null; then echo "pip not found, please run apt-get install python-pip"; exit 1; fi
	@if ! which virtualenv > /dev/null; then echo "virtualenv not found, please run apt-get install python-virtualenv"; exit 1; fi
	(unset PYTHONPATH; virtualenv ../build/doc_env; source ../build/doc_env/$(ACTIVATE); pip install --find-links=http://dnanexus-pypi2.s3.amazonaws.com/index.html --requirement=../build/doc_build_requirements.txt)
	virtualenv --relocatable ../build/doc_env
	touch .setup_doc_virtualenv

doc_python: python setup_doc_virtualenv
	rm -rf ../doc/python/*
	source ../build/doc_env/$(ACTIVATE); source "../environment"; export PYTHONPATH="$${PYTHONPATH}:../lib"; $(MAKE) -C python/doc html

doc_dxcpp:
	mkdir -p ../doc/cpp/dxcpp
	doxygen cpp/dxcpp/Doxyfile

doc_dxjson:
	mkdir -p ../doc/cpp/dxjson
	doxygen cpp/dxjson/Doxyfile

doc_java: api_wrappers
	mkdir -p ../doc/java
	(cd java; mvn compile javadoc:javadoc)

doc_R: R
	(export R_LIBS_SITE="$(DNANEXUS_HOME)/lib/R"; R --no-save < "$(DNANEXUS_HOME)/src/R/install_dxR_doc_deps.R")
	(export R_LIBS_SITE="$(DNANEXUS_HOME)/lib/R"; R -q -e "library('parser'); library('highlight'); library('staticdocs'); build_package('R/dxR', '$(DNANEXUS_HOME)/doc/R/')")

# Upload Agent dependencies
# =========================
ua_deps: openssl boost curl file/stage/lib/libmagic.la

UBUNTU_MIRROR=http://archive.ubuntu.com/ubuntu

shasum:
	if ! hash shasum; then cd "${DNANEXUS_HOME}/build/bin"; ln -s "$$(command -v sha1sum)" shasum; fi

git: shasum
	if ! hash git; then $(MAKE) build_git; fi

GIT_PV=1.8.4
GIT_SHA=6b3924a5e7a05893670ca9744b7f5166cf03193b
build_git:
	mkdir -p git
	curl -L "https://github.com/git/git/archive/v$(GIT_PV).tar.gz" > git-$(GIT_PV).tar.gz
	[[ $$(shasum git-$(GIT_PV).tar.gz|cut -f 1 -d ' ') == $(GIT_SHA) ]]
	tar -xzf git-$(GIT_PV).tar.gz -C git --strip-components=1
	(cd git && autoreconf --install && ./configure && $(MAKE) && $(MAKE) install)
	(cd ../build/bin && ln -s ../../src/git/git)

OPENSSL_PV=1.0.2g
OPENSSL_SHA=36af23887402a5ea4ebef91df8e61654906f58f2
openssl: shasum
	if [[ "$(OS)" == Windows_NT || "$(UNAME)" == Darwin || "$(CENTOS_MAJOR_VERSION)" == "5" || "$(RHEL_MAJOR_VERSION)" == "7" ]]; then \
	    $(MAKE) build_openssl; \
	fi

build_openssl: openssl/libssl.a

openssl/libssl.a:
	mkdir -p openssl
	curl ${UBUNTU_MIRROR}/pool/main/o/openssl/openssl_${OPENSSL_PV}.orig.tar.gz > openssl-${OPENSSL_PV}.tar.gz
	[[ $$(shasum openssl-${OPENSSL_PV}.tar.gz|cut -f 1 -d ' ') == $(OPENSSL_SHA) ]]
	tar -xzf openssl-${OPENSSL_PV}.tar.gz -C openssl --strip-components=1
ifeq ($(UNAME), Darwin)
	cd openssl; ./Configure darwin64-x86_64-cc -shared --openssldir="$$(pwd)/stage"
else
	cd openssl; ./config shared --openssldir="$$(pwd)/stage"
endif
	$(MAKE) -C openssl -j1 install

BOOST_LIBS=filesystem,program_options,regex,system,thread,atomic,chrono,date_time,exception,iostreams,signals

ifeq ($(UNAME)$(CC), Darwinclang)
	BOOST_MAJOR=1
	BOOST_MINOR=56
	BOOST_PV=$(BOOST_MAJOR).$(BOOST_MINOR)
	BOOST_URL=http://downloads.sourceforge.net/project/boost/boost/${BOOST_PV}.0/boost_${BOOST_MAJOR}_${BOOST_MINOR}_0.tar.bz2
	BOOST_SHA=f94bb008900ed5ba1994a1072140590784b9b5df
else
	BOOST_PV=1.55
	BOOST_URL=${UBUNTU_MIRROR}/pool/universe/b/boost${BOOST_PV}/boost${BOOST_PV}_${BOOST_PV}.0.orig.tar.bz2
	BOOST_SHA=cef9a0cc7084b1d639e06cd3bc34e4251524c840
endif

boost: boost/stage

boost-$(BOOST_PV).tar.bz2:
	curl -L ${BOOST_URL} > boost-$(BOOST_PV).tar.bz2

boost.download-shasum-checked: boost-$(BOOST_PV).tar.bz2 shasum
	[[ $$(shasum boost-$(BOOST_PV).tar.bz2|cut -f 1 -d ' ') == $(BOOST_SHA) ]]
	touch boost.download-shasum-checked

# See https://svn.boost.org/trac/boost/ticket/5680 for Windows/MinGW details
boost/stage: boost.download-shasum-checked
	mkdir -p boost
	tar -xjf boost-$(BOOST_PV).tar.bz2 -C boost --strip-components=1
ifeq ($(OS), Windows_NT)
	cd boost; ./bootstrap.sh --with-toolset=mingw --with-libraries=${BOOST_LIBS}
	sed -i -e s/mingw/gcc/ boost/project-config.jam
	cd boost; ./b2 --build-type=complete --layout=tagged -j8 stage
else ifeq ($(CENTOS_MAJOR_VERSION), 5)
	echo 'using gcc : 4.4 : g++44 ;' >> boost/tools/build/v2/user-config.jam
	$(MAKE) boost_build
else ifeq ($(UNAME)$(CC), Darwinclang)
	cd boost; ./bootstrap.sh --with-libraries=${BOOST_LIBS} --with-toolset=clang
	cd boost; ./b2 --layout=tagged -j8 stage cxxflags="-stdlib=libc++" linkflags="-stdlib=libc++" toolset=clang
else
	$(MAKE) boost_build
endif

boost_build:
	cd boost; ./bootstrap.sh --with-libraries=${BOOST_LIBS}
	cd boost; ./b2 --layout=tagged -j8 stage

curl: c-ares/stage/lib/libcares.la curl/stage/lib/libcurl.la

CURL_MAJOR=7
CURL_MINOR=45
CURL_MICRO=0
CURL_PV=$(CURL_MAJOR).$(CURL_MINOR).$(CURL_MICRO)
CURL_PV_UNDERSCORES=$(CURL_MAJOR)_$(CURL_MINOR)_$(CURL_MICRO)
CURL_MIRROR=https://github.com/bagder/curl/releases/download
CURL_URL=$(CURL_MIRROR)/curl-$(CURL_PV_UNDERSCORES)/curl-$(CURL_PV).tar.gz
CURL_SHA=e44fc290844c1e3c9b7f6ef9926d4366d0600514
ifeq ($(OS), Windows_NT)
	CURL_WITH_SSL=--with-winssl
	CURL_SHARED_LIBS=--enable-shared=yes
else ifeq ($(UNAME), Darwin)
	CURL_WITH_SSL=--with-ssl=$$(pwd)/../openssl/stage
	CURL_SHARED_LIBS=--enable-shared=yes
else ifeq ($(CENTOS_MAJOR_VERSION), 5)
	CURL_WITH_SSL=--with-ssl=$$(pwd)/../openssl/stage
	CURL_SHARED_LIBS=--enable-shared=no
else ifeq ($(RHEL_MAJOR_VERSION), 7)
	CURL_WITH_SSL=--with-ssl=$$(pwd)/../openssl/stage
	CURL_SHARED_LIBS=--enable-shared=no
else
	CURL_WITH_SSL=--with-ssl
	CURL_SHARED_LIBS=--enable-shared=no
endif
curl/stage/lib/libcurl.la: shasum openssl
	mkdir -p curl
	curl -L $(CURL_URL) > curl-$(CURL_PV).tar.gz
	[[ $$(shasum curl-$(CURL_PV).tar.gz|cut -f 1 -d ' ') == $(CURL_SHA) ]]
	tar -xzf curl-$(CURL_PV).tar.gz -C curl --strip-components=1
	cd curl; ./configure --disable-ldap --disable-ldaps \
	    --disable-rtsp --disable-dict --disable-telnet --disable-tftp --disable-pop3 \
	    --disable-imap --disable-smtp --disable-gopher --disable-sspi --disable-ntlm-wb \
	    --disable-tls-srp --without-gnutls --without-polarssl --without-cyassl \
	    --without-nss --without-libmetalink --without-libssh2 --without-librtmp \
	    --without-winidn --without-libidn --enable-ares="$(DNANEXUS_HOME)/src/c-ares/stage" \
	    --enable-static=yes $(CURL_SHARED_LIBS) $(CURL_WITH_SSL) --prefix=/
	$(MAKE) -C curl install DESTDIR="$(DNANEXUS_HOME)/src/curl/stage"

CARES_PV=1.13.0
CARES_SHA=dde50284cc3d505fb2463ff6276e61d5531b1d68
c-ares/stage/lib/libcares.la: shasum
	mkdir -p c-ares
	curl https://c-ares.haxx.se/download/c-ares-${CARES_PV}.tar.gz > c-ares-${CARES_PV}.tar.gz
	[[ $$(shasum c-ares-${CARES_PV}.tar.gz|cut -f 1 -d ' ') == $(CARES_SHA) ]]
	tar -xzf c-ares-${CARES_PV}.tar.gz -C c-ares --strip-components=1
	cd c-ares; ./configure --prefix=/
	$(MAKE) -C c-ares install DESTDIR="$(DNANEXUS_HOME)/src/c-ares/stage"

FILE_PV=5.32
FILE_SHA=6a84bddca6fa9bcf6882832ccece38cf22b9b600
file/stage/lib/libmagic.la: shasum
	mkdir -p file
	curl ${UBUNTU_MIRROR}/pool/main/f/file/file_${FILE_PV}.orig.tar.xz > file-${FILE_PV}.tar.xz
	[[ $$(shasum file-${FILE_PV}.tar.xz|cut -f 1 -d ' ') == $(FILE_SHA) ]]
	tar -xf file-${FILE_PV}.tar.xz -C file --strip-components=1
	cd file; ./configure --enable-static=yes --prefix=/
	$(MAKE) -C file install DESTDIR="$(DNANEXUS_HOME)/src/file/stage"

# Bundled utilities
# =================

jq: git_submodules ../bin/jq

../bin/jq: jq/
	(cd jq; autoreconf --install)
	(if [[ "$(UNAME)" == Darwin ]] && [[ ! "$(CC)" == clang ]] && ! (gcc --version | grep -q LLVM); then export LDFLAGS="$$LDFLAGS -lgcc_eh"; fi; \
	 cd jq; ./configure --disable-docs)
ifeq ($(OS), Windows_NT)
	$(MAKE) -C jq -j 1
else
	$(MAKE) -C jq -j PATH=$$(brew --prefix bison || echo .)/bin:$$PATH
endif
	cp -f jq/jq ../bin/

# System dependencies
# ===================

install_sysdeps:
ifeq ($(CENTOS_MAJOR_VERSION), 5)
	${MAYBE_SUDO} yum groupinstall -y "Development tools"
	${MAYBE_SUDO} yum install -y cmake gcc44-c++ zlib-devel bzip2-devel curl-devel
else ifeq ($(FEDORA_MAJOR_VERSION), 20)
	${MAYBE_SUDO} yum groupinstall -y "Development tools"
	${MAYBE_SUDO} yum install -y gcc gcc-c++ automake bison flex python python-pip python-virtualenv boost-devel boost-static cmake openssl-devel libcurl-devel bzip2-devel curl
else ifeq ($(UBUNTU_VERSION), 12.04)
	${MAYBE_SUDO} apt-get install make python-setuptools python-pip python-virtualenv python-dev \
		g++ cmake libcurl4-openssl-dev zlib1g-dev libbz2-dev flex bison autoconf curl
else ifeq ($(UBUNTU_VERSION), 14.04)
	${MAYBE_SUDO} apt-get install make python-setuptools python-pip python-virtualenv python-dev \
		g++ cmake libcurl4-openssl-dev zlib1g-dev libbz2-dev flex bison autoconf curl
endif

# Clean
# =====

distclean: clean

clean:
	$(MAKE) -C dx-verify-file clean
	$(MAKE) -C ua clean
	-find ../bin -type f \( -name jq -or -name dx -or -name 'dx-*' \) -not -name 'dx-unpack*' -delete
	-rm -f ../bin/docker2aci ../bin/proot
	-rm -rf python/{build,*.{egg,egg-info}}
	-rm -rf java/target
	-rm -rf ../lib
	-rm -rf ../share
	-rm ../build/info/version
	-rm -rf "$(DX_PY_ENV)"
	-rm -rf boost c-ares curl file openssl
	-$(MAKE) -C jq clean

.PHONY: all toolkit_version api_wrappers src_libs python pynsist_installer java ua test test_python install base_install debian_install doc setup_doc_build_virtualenv doc_python doc_dxcpp doc_dxjson doc_java R doc_R boost boost_build install_sysdeps
