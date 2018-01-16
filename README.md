# Setup a python package

## System requirements
Build tool
Compiler
Make - osx/linux
Windows - unknown make requirement

## Notes OSX

Missing components
    openssl
    readline
 
## Notes on Linux

Missing components
    openssl
    readline

create centos docker
yum install openssl-devel readline-devel

## Notes on Windows

Because it is not trivial to build Python 2.7 download msi package and
repackage the binaries after adding the requirements.
