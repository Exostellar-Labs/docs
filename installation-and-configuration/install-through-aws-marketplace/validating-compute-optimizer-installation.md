# Validating Compute Optimizer Installation

Run the following command to validate the Compute Optimizer:

```
xspot check -f
```

{% hint style="success" %}
The Compute Optimizer should pass all prerequisites as shown below:
{% endhint %}

<figure><img src="../../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

Turn the scheduler off:

```
xspot scheduler off
```

Request an on-demand instance from the Compute Optimizer Controller:

```
xspot add -c 4 -m 8192
```

{% hint style="success" %}
Sample success output for `xspot ps`:
{% endhint %}

<figure><img src="../../.gitbook/assets/Screenshot 2023-11-10 at 10.02.16 AM.png" alt=""><figcaption></figcaption></figure>
