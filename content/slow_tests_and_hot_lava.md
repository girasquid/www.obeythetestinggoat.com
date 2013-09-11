Title: Fast tests are useless, hot lava be damned
Date: 2013-09-11 04:00
Tags: unit tests, speed, mocking
Slug: fast-tests-useless-hot-lava-be-damned
Author: Harry
Status: draft
Summary: <p>In which I troll the "fast tests" community by claiming they are useless</p>

tl;dr: I think it's a waste of time to strive for an ultrafast test suite,
because it leads to overly mocky, disjointed tests which don't help you
catch bugs and don't help you code well.  Tell me why I'm wrong!

At this year's DjangoCon, Casey Kinsley gave a talk in which he ostensibly
advocated striving for faster test suites.  As we'll find out, I've often
thought that stiving for fast tests suites can be a bit misguided, so I went
along preparing to disagree loudly -- as anyone who knows me can attest,
disagreeing loudly is just about my favourite thing to do, and a major
character flaw.

As it turns out I found very little to disagree with, aside from [one
particularly tortured use of a Mock object](http://www.slideshare.net/cordiskinsey/djangocon-2013-how-to-write-fast-and-efficient-unit-tests-in-django/22)
which definitely made baby test sanity Jesus cry.  It was an excellent talk, 
and full of good advice, as well as the memorable quote "The Database is Hot
Lava"

## The traditional argument: slow tests are bad

So what's going on?  Here's the traditional argument for fast tests:

Tests that take ages to run won't get run, which will break your development
process in several ways, and reduce the benefits you get from testing.  You
won't pick up defects early, because no-one is running the full test suite
before committing, and you won't be able to get the secondary benefit which 
you hope to gain from TDD, which is better designed code, because you can't
do TDD with slow tests.

The reference for this point of view is Gary Berhnahardt's talk from Pycon
2012, entitled [Fast Tests, Slow test](https://www.youtube.com/watch?v=RAxiiRPHS9k)
If you haven't already, I definitely encourage you to watch that talk, Gary 
knows what he's talking about, whereas I suspect I don't.

## Why I think this is wrong: fast tests don't help you code, and they don't help you find bugs

I'm being deliberately provocative here, and this is a bit of a strawman, but
I think the problem with turning slow tests into fast tests is that you end
up with worse tests.  Why is that?

* faster tests mean more "unittey" tests -- tests with less dependencies
* that means using more mocks
* that leads to a very granular tests, which are highyly decoupled from each
other
* and that, ultimately, doesn't help you find unexpected bugs
* and it doesn't help you find regressions when you're refactoring either

So let's look at an example. Imagine our site does a bit of setup for each user:
it makes a temp folder for them, and then sets a flag on their user profile:

Imagine this:

    :::python
    def setup_user_environment(user):
        setup_temp_storage(user)
        profile = user.get_profile()
        profile.environment_setup = True
        profile.save()

Here's the kind of test I'm inclinde to write:

    :::python
    def test_setup_user_environment_sets_up_temp_storage_and_sets_flag_on_profile(self):
        user = User.objects.create(username='jim')
        setup_user_environment(mock_user)
        self.assertTrue(os.path.exists('/tmp/jim')
        self.assertTrue(user.get_profile().environment_setup)

Arg!  That's not a unit test at all!  It touches the database, which is hot lava! Worse
still, it touches the filesystem!

Here's the ultimate "fast" test for this:

    :::python
    def test_setup_user_environment_sets_up_temp_storage_and_sets_flag_on_profile(self):
        mock_user = Mock()
        with patch('myapp.setup_temp_storage') as mock_setup_temp_storage:
            setup_user_environment(mock_user)
        mock_setup_temp_storage.assert_called_once_with(mock_user)
        mock_profile = mock_user.get_profile.return_value
        self.assertEqual(mock_profile.environment_setup, True)

Well, I would argue that you have a much less readable test there, and it's also a test
that's very closely coupled to the implementation.  It discourages refactoring, because
something as simple as changing the name of the helper method `setup_temp_storage` involves
changing the test code in 4 places -- two of which (`mock_setup_temp_storage`) won't be found
by automated refactoring tools.

Or, imagine I change `setup_temp_storage` to take a username instead of a user.  I go and find
its unit tests and change it, then change its implementation.  What will happen next is that
my unit test for `setup_user_environment` will break, because it uses the real function, and
so that's my reminder to change the place it gets used. 

In contrast, in the fast test, `setup_user_environment` is mocked, so that test will still pass,
even though my code is broken.

Sure you could argue that my `os.path.exists` call is tightly coupled to the implementation of
`setup_temp_storage`, and that if I end up using it in lots of tests, they'll be annoying to 
change, if I ever change the location of temp storage, for example.  But I could factor it out
into a test helper method, if I notice myself duplicating test code a lot.

Now unit test purists and I would probably agree that this example doesn't prove you should
*never* mock anything, or that "proper" unit tests are useless.  What I am saying is that,
in a case where you can test a piece of code with either a mocky or a non-mocky test, I prefer
non-mocky tests.  Gary Berhnardt says you should aim for 90% unit tests vs 10% "integration" tests
(which is what you'd call the kind of test I write), and I tend to think the ratio is more like
50/50.

Unit tests are definitely better for nailing down code which has lots of edge cases and possible
logical pitfalls -- like the classic example of a roman numeral converter.  But my assertion is:
in web development, that kind of code is rare:

- helper methods on models tend to be quite simple
- view functions tend to be simple:  get a post request, instantiate a form,
  save a database item if the form validates and redirect, return a template
  with errors if not
- code tends to be hard to unit test without mocking -- the database, the
  Django request/response stack, or the template layer, or whatever.

I've just tended to find that there aren't many places where I find myself spelling out more than
2 or 3 tests for any given function -- in which case, I tend to find, unit tests don't offer
any substantial advantage.  In the example above, there's probably only one two cases, maybe a 
second one for the case where `setup_temp_storage` raises an exception.

But what about the fact that integration tests are slow?  The database is hot lava!  Isn't it?

## The database is, at worst, lukewarm lava

Remember, Django uses an *in-memory* Sqlite database when you're running its unit tests. It's
pretty fast.  Here's a test suite which hits the DB for some tests and not for others, with
1,000 tests of each:

    :::python
    import unittest
    from django.test import TestCase
    from myapp.models import Car

    class FastTest(unittest.TestCase):
        def create_car(self):
            return Car(colour = 'Blue')

    class SlowTest(TestCase):
        def create_car(self):
            return Car.objects.create(colour='blue')

    for i in range(1000):
        method_name = 'test_car_{0}'.format(i)
        def testfn(self):
            car = self.create_car()
            self.assertEqual(car.colour, 'Blue')
        setattr(FastTest, method_name, testfn)
        setattr(SlowTest, method_name, testfn)


So what's the difference in speed between the two?

    $ python manage.py test myapp.FastTest
    Ran 1000 tests in 0.032s

    $ python manage.py test myapp.SlowTest
    Ran 1000 tests in 0.231s

A factor of 8.  An order of magnitude perhaps, but not two orders of magnitude.
And notice that's 1000 tests, still running in less than a second. So, is shifting
from tests that run in microseconds to tests that run in tens of microseconds *really worth*
all the losses in terms of readability and ease of refactoring?


