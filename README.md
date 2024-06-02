This repository contains manifests for building [Torizon OS](https://developer.toradex.com/torizon/) (formerly TorizonCore) and Toradex Reference images.
It was forked from https://git.toradex.com/cgit/toradex-manifest.git/ in order to make adaptations for the Google Summer of Code 2024, particularly to support the following projects:

- [Uptane: Aktualizr Integration With SWUpdate](https://summerofcode.withgoogle.com/programs/2024/projects/qQntgfyx)
- [Building support for an A/B partition scheme-based update for Uptane Client](https://summerofcode.withgoogle.com/programs/2024/projects/8r2exO5W)

# Usage instructions

- Install the Google repo tool:
  ```
  $ mkdir ~/bin
  $ PATH=~/bin:$PATH

  $ curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
  $ chmod a+x ~/bin/repo
  ```

- Make sure your Git user name and e-mail are configured correctly:
  ```
  $ git config --global user.email "you@example.com"
  $ git config --global user.name "Your Name"
  ```

- Check out the metadata for GSoC:
  ```
  $ mkdir gsoc-2024
  $ cd gsoc-2024
  $ repo init -u https://github.com/rborn-tx/gsoc-manifest -b gsoc-2024 -m gsoc.xml
  $ repo sync -j8
  ```
