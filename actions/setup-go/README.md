# MKMBA setup-go wrapper

This action wraps the top-level actions/setup-go GitHub action to provide the following functionality:

1) A default MKMBA production go language version that is used by default.
2) The ability for a caller to pass the location of a Dockerfile from which a go language version should be extracted

Once the action has found a go language version to use, it calls the stnadard setup-go action to install it, and then
sets GOTOOLCHAIN=local into the environment for later steps so that the installed version is used without any further
changes.
