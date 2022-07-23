# How to build

* Install Ubuntu 22.04, ideally in a VM. We will run everything inside that system.


## Install docker
```sh
sudo apt install docker.io
# sudo groupadd docker # this is done automatically on ubuntu
sudo usermod -aG docker $USER
# if using virtualbox, set up some permissions for the user
sudo usermod -aG vboxsf $USER
# log out, log in to apply permissions to the user
```


## Build buildozer docker image

sadly the kivy/buildozer is very outdated and does not work with latest python-for-android,
so we'll have to build our own.

```sh
# install requirements
sudo apt install git
# checkout and build the latest release of buildozer as docker image
git clone https://github.com/kivy/buildozer.git
git checkout 1.4.0
docker build --tag=buildozer .
# now we create a blank project to pull the default android sdk and ndk stuff
# this will download some extra into the blank project, and compile a lot, but with ccache
# (to save bandwidth this could be done with the actual project instead)
mkdir blank-project
cd blank-project
touch main.py
docker run --volume "$(pwd)":/home/user/hostcwd buildozer init
docker run -it --name new_buildozer --volume "$(pwd)":/home/user/hostcwd buildozer android debug
# name the new docker image based on the now-installed NDK. We will use this from now on.
docker commit new_buildozer buildozer-r23b
docker rm new_buildozer
# we now have a docker with sdk, ndk, tools and maybe some level of ccache
# we can create a script that allows us to simply use "buildizer-r23b" from now on
echo 'docker run --volume "$(pwd)":/home/user/hostcwd buildozer-r23b' > ~/.local/bin/buildozer-r23b
chmod a+rx ~/.local/bin/buildozer-r23b
echo "PATH=~/.local/bin:$PATH" >> ~/.bashrc
export "PATH=~/.local/bin:$PATH"
```


## CommonClient

### Check out the source and customize it
```sh
git clone https://github.com/ArchipelagoMW/Archipelago.git
cd Archipelago
# remove Main.py (to avoid confusion)
rm Main.py
# make source folder is identified as source package, not a setuptools package
touch __init__.py
rm setup.py
# rename entry point to main.py and make AP a package
mv CommonClient.py main.py
# remove stuff we don't want included; can also be done via buildozer.spec below
rm -R test docs WebHostLib *Adjuster.py *Client.py Fill.py Generate.py Launcher.py WebHost.py
mv worlds worlds.all
mkdir worlds
mv worlds.all/*.py worlds.all/generic worlds/
rm -R worlds.all
# TODO: exclude sprites!!!
# TODO: make sure to exclude unused stuff from data/
# init android project
buildozer-r23b init
```

* modify buildozer.spec here - set requirements and what not.
  the final .spec should be in a repo, so that has to be done only once
  * `android.permissions = INTERNET,ACCESS_NETWORK_STATE`
  * `requirements = python3,kivy,colorama,websockets,pyyaml,schema`
  * jellyfish would require a recipe, that does not exist, so we'll make it optional below
* if manifest needs to be customized beyond buildozer.spec, change the template in
  `.buildozer/android/platform/build-*/dists/*/templates/AndroidManifest.tmpl.xml`
* add android detection to `ModuleUpdate.py`:

    ```python
    # after update_ran = getattr(...) add
    try:
        import android
        update_ran = True
    except:
        pass
    ```

* make jellyfish optional in `Utils.py`

    ```diff
    -import jellyfish
    +try:
    +    import jellyfish
    +except:
    +    jellyfish = None
    ```
    ```diff
     def get_fuzzy_ratio(word1: src, word2: src) -> float:
    +    if not jellyfish:
    +        return 0
    ```

### Build the source
```sh
# build the android project
buildozer-r23b android debug
```

### Addendum: requirements
```
websockets
colorama  # (via MultiServer.py, interesting)
websockets
PyYaml  # (via Utils.py)
schema # (via worlds/, could be removed)
# jellyfish  # (via Utils.py, we disable it)
```


