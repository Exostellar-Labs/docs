# Script Deployment

> **Note:**
>
> You should be :

| Who? | On which system?  | In which directory?             |
| ---- | ----------------- | ------------------------------- |
| root | X-Spot controller | /nfs-apps/exostellar/onboarding |

> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

1.  Before deploying to production, check that `/etc/xspot` configuration assets are synced with the source onboarding directory:

    ```
    make sync-config-preview
    ```

    * If any important config files are showing discrepancies, the `diff` will be displayed.
2.  To synchronize assets in the source onboarding directory based on the working configurations in `/etc/xspot`, run the following command:

    ```
    make sync-config
    ```

    * You can now proceed to releasing the assets to production in the next step.
3.  Take a snapshot of the `onboarding` folder in its current state as a safeguard and finalize the recently validated image for use running jobs with X-Spot:

    ```
    make deploy
    ```
4. As a final validation, restart the X-Spot suite of services on the X-Spot controller.
   *   To faciliate, make use of X-Spot's `restart` command:

       ```
       xspot restart
       ```
5.  (Optional) Rerun a quick validation workload or retest X-Spot interactively:

    ```
    exorun run -c 4 -m 8 -i docker-base -- /bin/bash
    ```
