# firely

Firely is an A/B Testing overlay based on Firebase Remote Config.
It's a work in progress to simplify the integration and make the management of A/B testing XPs safer. 

## How does it work

This library, integrated in your gradle project, only requires:
- A `firely-config.json` file that will contains the type of items, the keys, and the default value
- A call to `Firely.setup(Context context)` from the `Application.onCreate()` method
- One proguard rule

`firely-config.json` file is organized in 3 main sections (for us, but it can have the "names" you want):
- Feature Flags
- Config
- Experiments (or A/B Tests)


Here is an example of `firely-config.json`:
```json
{
  "config": [
    {
      "key": "android_version_code_min",
      "default": 0
    }
  ],
  "feature_flag": [
    {
      "key": "refer_a_friend",
      "default": true
    },
    {
      "key": "promotion_url",
      "default": ""
    }
  ],
  "experiment": [
    {
      "key": "xp_button_pay",
      "default": "control"
    }
  ]
}
```

Firely is an Android library that come with a gradle plugin, `firely-plugin`. It will generate a `FirelyConfig.java` file based on the `firely-config.json`, like the `R.java` android creates. The `FirelyConfig.java` will contain Enums that match the configuration. You can then use these enums on Firely to get `LiveVariable`, `CodeBlock`, `OrderedArrayBlock`.


### LiveVariable

Let's imagine I am using Remote Config to restrict my user to a minimum Android version on which they can run (otherwise they have to update the app). With Firely, I can instantiate a LiveVariable that will use this setting:

```java
LiveVariable<Integer> minAndroidRemoteVersion = Firely.integerVariable(FirelyConfig.Config.ANDROID_VERSION_CODE_MIN);
```

`FirelyConfig.Config.ANDROID_VERSION_CODE_MIN` is generated by the plugin and the default value is 0.

Now, anytime I need to get the last version that has been fetched, I just call:

```java
Integer lastVersion = minAndroidRemoteVersion.get();
```

Here is another example with a feature flag:

```java
if (Firely.booleanVariable(FirelyConfig.FeatureFlag.REFER_A_FRIEND).get()) {
	// Add the view
}
```


### CodeBlock

Now I need to build out an XP that will change the text of a button.

```java
Firely.codeBlock(Remote.Experiment.XP_BUTTON)
	.withVariant("billed_currency", "no_price")
	.execute(
	() -> advance.setText(getString(R.string.bb_payment_cta)), // control
	() -> advance.setText(.getString(R.string.bb_payment_cta_2)), // billed_currency
	() -> advance.setText(getString(R.string.bb_payment_cta_3))); // no_price
```

> NOTE: we are always using "control" as the default value and as the control group for A/B Tests.


### OrderedArrayBlock

In the [Busbud Android App](https://play.google.com/store/apps/details?id=com.busbud.android), we use a lot of blocks and lists. Let's imagine you have *N* blocks of data in a page.
You want to A/B test which one should go first and the order for all the others.
A basic approach could be to have *N!* variants.

If we have three items: 1-2-3, 2-1-3, 2-3-1, 1-3-2, 3-2-1, 3-1-2

And while using CodeBlocks:

```java
Firely.codeBlock(Remote.Experiment.XP_BUTTON)
	.withVariant("2-1-3", "2-3-1", "1-3-2", "3-2-1", "3-1-2")
	.execute(
	() -> {
		addOne();
		addTwo();
		addThree();
	}, // control
	() -> {
		addTwo();
		addOne();
		addThree();
	},
	... etc

```
Really inefficient.

Another approach is to use OrderedArrayBlock. You will use one Firebase entry:

```json
{
  ...
  "experiment": [
    {
      "key": "xp_mypage_order",
      "default": "one,two,three"
    }
  ]
}
```

```java
OrderedArrayBlock mCheckoutXp = 
		Firely.orderedArrayBlock(FirelyConfig.Experiment.XP_CHECKOUT_ORDER)
			.addStep("one", () -> addOne())
			.addStep("two", () -> addTwo())
			.addStep("three", () -> addThree());
```

And you can control your A/B Tests from the Firebase Remote Config dashboard by changing the `xp_mypage_order` key.

`three,one,two` will then call `addThree()`, `addOne()`, `addTwo()`. You can use this to remotely control the order of lists.

## Analytics

One of the highlights of Firebase is that everything is working together. In the [documentation](https://firebase.google.com/docs/remote-config/config-analytics), Firebase proposes putting the values, manually, as a User Property:

```java
String experiment1_variant = FirebaseRemoteConfig.getInstance().getString("experiment1");
   AppMeasurement.getInstance(context).setUserProperty("MyExperiment",experiment1_variant);
```

That's nice, but it does not fit our needs. Putting the property at the user level means it will be erased over time and we will lose the information. Instead, we prefer to tag all the events with all the experiments that have been applied at the time the event is triggered.

We added a method on Firely to help with this:

```java
Firely.getAllPropsWithCurrentValue()
```
And this method is called each time we send an event and merged into the property list. 
Therefore we can track the configuration changes over time.


## Build locally

You can build locally the project by:

1. Building the plugin:

`./gradlew :firely-plugin:publishToMaven -c plugin.gradle`


2. Create a firebase project for `com.busbud.android.firely.sample` and copy the google-service.json in the /firely-sample repository

2. Build the project: `./gradlew assembleDebug`



