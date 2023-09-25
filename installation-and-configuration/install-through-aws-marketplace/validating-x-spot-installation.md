# Validating X-Spot Installation

Run the following command to validate the X-Spot:

```
xspot check -f
```

{% hint style="success" %}
The X-Spot should pass all prerequisites as shown below:
{% endhint %}

<figure><img src="../../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

Turn the scheduler off:

```
xspot scheduler off
```

Request an on-demand instance from the X-Spot Controller:

```
xspot add -c 4 -m 8192
```

{% hint style="success" %}
Sample success output for `xspot ps`:
{% endhint %}

<figure><img src="../../.gitbook/assets/image (12).png" alt=""><figcaption><p><code>xspot ps</code> output</p></figcaption></figure>

