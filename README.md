# FluentSelenium

Refer my [Fluent Selenium Examples Blog Entry](http://paulhammant.com/2013/05/19/fluent-selenium-examples)
about this, or the project that showcases Fluent Selenium - [Fluent Selenium Examples](https://github.com/paul-hammant/fluent-selenium-examples).

To use via Maven:

```xml
<dependency>
   <groupId>org.seleniumhq.selenium.fluent</groupId>
   <artifactId>fluent-selenium</artifactId>
   <version>1.8.1</version>
   <scope>test</scope>
</dependency>

<!-- you need to choose a hamcrest version that works for you too -->
<dependency>
   <groupId>org.hamcrest</groupId>
   <artifactId>hamcrest-all</artifactId>
   <version>1.3</version>
   <scope>test</scope>
</dependency>
```

For non Maven build systems, [download it yourself](http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22fluent-selenium%22)

## Basic Use

```java
WebDriver wd = new FirefoxDriver();
FluentWebDriver fwd = new FluentWebDriver(wd);

fwd.div(id("foo").div(className("bar").button().click();

fwd.span(id("results").getText().shouldBe("1 result");
```

## Situations where the DOM is changing slowly

### within()

There's a "within" capability in the fluent language. It will retry for an advised period,
giving the fluent expression a chance to get past a slowly appearing node:

```java
fwd.div(id("foo").div(className("bar").within(secs(5)).button().click();

fwd.span(id("results").within(millis(200)).getText().shouldBe("123");
```

### without()

There's a "without" capability in the fluent language. It will retry for an advised period,
giving the fluent expression observe that something in the page should disappear:

```java
fwd.div(id("foo").div(className("bar").without(secs(5)).button();
```

The element disappearing in the page means that the fluent expression stops
there. Also, disappear means that the locator used to find the element does
not find it, thus the following does not mean that there's no span element,
it just means that there is no span element with a class of "baz":

```java
fwd.div(id("foo").div(className("bar").without(secs(5)).span(className("baz"));
```


### Stale Elements

WebDriver, by default, does not handle findElement traversals from elements that have
gone stale transparently. It prefers to throw StaleElementReferenceException, which you
have to catch and then do something with. Retry is one option. FluentSelenium has retry
capability:

```java
new RetryAfterStaleElement() {
    public void toRetry() {
        div(className("fromto-column")).getText().toString();
    }
}.stopAfter(secs(8));
```

In this example, the element can go stale any amount of times in eight seconds, and the whole
traversal is restarted again and again.  If you're trying to store values, you'll have a
problem with Java's inner-class rules, and have to do dirty tricks like:

```java
final String selectedFlight[] = new String[1];
new RetryAfterStaleElement() {
    public void toRetry() {
        selectedFlight[0] = div(className("fromto-column")).getText().toString();
    }
}.stopAfter(secs(8));
```
Use of the one elem array is the dirty trick, because of the need for final.

## Built-in Assertions

### Strings

Many things return a string (actually a TestableString). Some elements of a page
are designed to have a string representation.  Input fields and spans are obvious,
but any element supports getText() and WebDriver will try to make a chunk of text
that represents that (often with carriage returns).

```java
fwd.div(id("foo").getText().shouldBe("1 bar");
fwd.div(id("foo").getText().shouldNotBe("0 bars");
fwd.div(id("foo").getText().shouldContain("bar");
fwd.div(id("foo").getText().shouldNotContain("error");
```

Regex is possible too, and it will ignore carriage returns (which Java pre-processes like so \n -> \\n)

```java
fwd.div(id("foo").getText().shouldMatch("\d bar");
fwd.div(id("foo").getText().shouldMatch("[1-9] bar");
fwd.div(id("formErrors").getText().shouldNotMatch("\d errors");
```

As shown above, you can transparently wait for the think to become true:

```java
fwd.div(id("foo").getText().within(secs(10)).shouldBe("1 bar");
```

If used in conjunction with a within(..) expression the assertion is also retried
subject to the advised period.

### Others

Any element has a location via getLocation(), which yields a Point
Any element has a size via getSize(), which yields a Dimension
Some elements have boolean from isDisplayed(), isEnabled() and isSelected()

All of these have assertions:

```java
fwd.div(id("foo").getLocation().shouldBe(new Point(1, 1));
fwd.div(id("foo").getLocation().shouldNotBe(new Point(1, 1));

fwd.div(id("foo").getSize().shouldBe(new Dimension(640, 480));
fwd.div(id("foo").getSize().shouldNotBe(new Dimension(640, 480));

fwd.div(id("foo").isEnabled().shouldBe(true);
fwd.div(id("foo").isDisplayed().shouldNotBe(false);
```

## Locators

WebDriver's own "By" locator mechanism is what is used. Here are examples using that:

```java
By.id("id")
By.className("name")
By.tagName("table")
```

Class FluentBy adds a few more:

```java
FluentBy.attribute("ng-model")
FluentBy.attribute("ng-model", "shopperSelection.payPalPreferred") {
FluentBy.composite(tagName("table"), className("paymentType"))
FluentBy.composite(tagName("table"), attribute("ng-click")) {
```

One more strictClassName is used like so:

```java
FluentBy.strictClassName("name")
```

Strict is where there is only one class for that element. The built-in WebDriver one allows
for many classes for an element, with the one specified amongst them.

# Multiple elements

Just like WebDriver, FluentSelenium can return a collection of Elements matching a locator:

```java
FluentWebElements elems = fwd.div(id("foo").div(className("bar").buttons();
elems = fwd.div(id("foo").divs(className("bar");
elems = fwd.divs(id("foo");
```

Look at the pluralization of the methods above, and that it only makes sense if
it's the last in the fluent expression.

## Fluently traversing through multiple elements:

Use a FluentMatcher instance (which is just a predicate)

```java
FluentMatcher fm = new MyIntricateFluentMatcher();
// click on first matching one...
fwd.inputs(className("bar").first(fm).click();

// click on all matching matching ones...
fwd.inputs(className("bar").filter(fm).click();
```

There are no instances of FluentMatcher built in, other than CompositeFluentMatcher.

# Exceptions

Obviously you want tests using FluentSelenium to pass.  Getting tests to be stable has also been a
historical challenge for the Selenium world, but a real failure of previously working test, is worth
debugging (before or after a developer commit that may have broken the build).

Fluent-Selenium throws exceptions that show fluent context for WebDriverException? root causes like so:

```
      "WebDriver exception during invocation of : ?.div(By.className: item-treasury-info-box')).h3()"
```

That exception's <code>getCause()</code> will be the WebDriverException derivative that happened during
the <code>h3()</code> invocation -  implicitly before any subsequent operation like click().
