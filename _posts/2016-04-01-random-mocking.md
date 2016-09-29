---
layout: post
title: Randomness Mocking With JMockit
---

If you had already tried to cover same bigger part of your code with JUnit tests, you have probably encountered it.
With JUnit we can run tested code with predefined input and check output of it.
But what if this code use, on background without control of calling code, same objects, which provide non-deterministic or changing output, like random numbers generator or real time source?
How can we cover such code with tests? This can be solved using mocking.

Mocking is creating fake objects, which will act role of the object, which we want to get out of the test.
Such a fake objects will be spoofed on place it and tested code will use within our test fake object instead of unpleasant real objects.

In this tutorial I will show example, how to mock most problematic classes of JDK.

At first we need to add JMockit library into us project. If you use maven, it is simple - just add following dependency:

{% highlight xml %}
        <dependency>
            <groupId>org.jmockit</groupId>
            <artifactId>jmockit</artifactId>
            <version>1.10</version>
            <scope>test</scope>
        </dependency>
{% endhighlight %}

As the second step we need to ensure that the test will be started with JMockit:

{% highlight java %}
@RunWith(JMockit.class)
public class MyRandomMockingTest {
{% endhighlight %}

Before mocking `Random` itself, lets try it with much simpler, but closely related, subject - mocking of system time.
We will prepare mock of `System` class - here we would like to ensure constant "current" time:

{% highlight java %}
    public static class SystemMock extends MockUp<System> {
        @Mock
        public long currentTimeMillis(){
            return 123;
        }
        @Mock
        public long nanoTime(){
            return 1234;
        }
    }
{% endhighlight %}

Just note - for mocking of `Random`/`SecureRandom` is not mocking of system time required. But it is not only example too - if you want to mock `Random`, it is likely you will want to mock the system time too.

Similar mock as for the `System` class we will prepare to mock the `Random` class. But it can cause problems in same algorithms (like key generation), where returning of constant "random" values everytime would cause getting stuck in infinite loop. We will keep pseudorandomicity of the `Random` output, but we will ensure that it will return the same pseudorandom sequence on every test run. Following code will replace seed (from which the pseudorandom numbers are generated) on every new instance of `Random` creation:

{% highlight java %}
    public static class RandomMock extends MockUp<Random> {
        @Mock
        public void $init(Invocation inv) throws Exception {
            Field field = Random.class.getDeclaredField("seed");
            field.setAccessible(true);
            field.set(inv.getInvokedInstance(), new AtomicLong(7326906125774241L));
        }
    }
{% endhighlight %}

We will keep all methods of `Random` untouched - they generate random values based on the seed, which is what we want.

If we would like to mock the `SecureRandom` too, we just replace it with non-secure `Random`, which is already mocked:

{% highlight java %}
    public static class SecureRandomMock extends MockUp<SecureRandom> {
        Random random = new Random();
        @Mock
        public void nextBytes(byte[] bytes){
            random.nextBytes(bytes);
        }
    }
{% endhighlight %}

Other methods of `SecureRandom` use this method to obtain the random values, so we can keep them as they are.

To start use defined mock classes we have to create new instances of them. In JUnit we can use method with `@BeforeClass` annotation to ensure start using them before every test run:

{% highlight java %}
    @BeforeClass
    public static void installMockClasses() throws Exception {
        new SystemMock();
        new RandomMock();
        new SecureRandomMock();
    }
{% endhighlight %}

The tests alone will look as usual, just mocked classes will provide mocked output:

{% highlight java %}
    @Test
    public void testMocking() {
        Assert.assertEquals(123, System.currentTimeMillis());
        Assert.assertEquals(1234, System.nanoTime());

        Random random = new Random();
        Assert.assertEquals(-1044329672, random.nextInt());
        Assert.assertEquals(2005169516, random.nextInt());

        SecureRandom secureRandom = new SecureRandom();
        Assert.assertEquals(952877249, secureRandom.nextInt());
        Assert.assertEquals(1819640951, secureRandom.nextInt());
    }
{% endhighlight %}

The complete example of above you can found in [gist](https://gist.github.com/honza889/67b4cb84a7bd5a345c8f0914c73c72c6).

