# Backup or Archive

* We recommend relocating, renaming, or archiving the previous `onboarding` folder to keep the work area tidy, but it is perfectly safe to destroy the previous `onboarding` directory entirely. This is because during the previous installation, the `make deploy` step did take a point-in-time snapshot of the directory.
  1.  Start in the `onboarding` directory.

      ```
      cd /nfs-apps/exostellar/onboarding
      ```
  2.  Create `git` "diff" file so we can easily track any customizations that currently exist.

      ```
      git diff > ../custom.diff
      ```
  3.  Make an `archive` directory and relocate the `onboarding` folder there.

      ```
      mkdir ../archive
      cd ..
      mv onboarding/ archive/onboarding.$( date +%s )
      ```
  4.  Download the latest tarball via the following command:

      ```
      pushd /tmp
      wget https://rpm.exostellar.io/download/onboarding.tar
      popd /tmp
      ```
  5.  Unpack the tarball.

      ```
      tar xvf /tmp/onboarding.tar
      ```
  6.  We should now have a fresh `onboarding` directory and an archive as follows:

      ```
      [root@x-infrastructure-optimizer-controller]# ls -la
      drwxr-xr-x   3 root  root    96 Jun 29 13:04 archive
      -rw-r--r--   1 root  root  1172 Jun 29 13:04 custom.diff
      drwxr-xr-x   9 root  root   288 Jun 29 13:04 onboarding

      [root@x-infrastructure-optimizer-controller]# ls -la archive/
      total 0
      drwxr-xr-x  3 root  root   96 Jun 29 13:04 .
      drwxr-xr-x  5 root  root  160 Jun 29 13:21 ..
      drwxr-xr-x  9 root  root  288 Jun 29 13:02 onboarding.1688058267
      ```
  7.  Navigate to the new `onboarding` directory.

      ```
      cd onboarding
      ```
  8. We will initialize the `git` version-control system again as before:
     *   Initialize an empty local repository:

         ```
         git init .
         ```
     *   Add everything in the directory to the `git` version-control system:

         ```
         git add .
         ```
     *   "Commit" the current state of the files in and under this directory so we can track changes.

         ```
         git commit $( cat version )
         ```
  9.  Now, we can bring forward out customizations from the previous working configurations:

      ```
      git apply --reject ../custom.diff
      ```
  10. You can inspect the applied changes and verify your customizatioins with another `git diff` now:

      ```
      git diff
      ```
  11. You can inspect the applied changes and verify your customizatioins with another `git diff` now:

      ```
      git diff
      ```
