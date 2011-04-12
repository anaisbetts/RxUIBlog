# ReactiveXaml: A compelling combination of MVVM and Reactive Extensions (Rx) 
	
I've been hacking on a library in my spare time
(Hah!) that I really think has the potential to change how folks
write Silverlight/WPF applications and I'm really excited about it.
After testing concepts and refining the interface using several
different sample apps, I believe I'm confident enough in the
concept that it needs to be seen by more people. I also have a
newfound respect for the folks who worked on the BCL and the PMs in
DevDiv - writing a library to get something done is easy; writing a
library that gets stuff done *and* is elegant and straightforward
is decidedly not. This library is an exploration I've been working
on for several weeks on combining WPF Model-View-ViewModel paradigm
with the Reactive Extensions for .NET (Rx). Combining these two
make managing concurrency as well as expressing complicated
interactions between objects possible in a declarative, functional
way. Put simply, if you've ever had to chain events / callbacks
together and declare state ints/booleans to keep track of what's
going on, Reactive Extensions provides a sane alternative. I'm
going to be posting quite a bit more about this library as well as
a sample application, but for now, check out
[the code to ReactiveXaml on Github](http://github.com/xpaulbettsx/ReactiveXaml).

What's in this library
----------------------

`ReactiveCommand` - an implementation of ICommand that is also a
Subject whose OnNext is raised when Execute is executed. Its
CanExecute can also be defined by an IObservable<bool\> which means
the UI will instantly update instead of implementations which rely
on RequerySuggested. `ReactiveAsyncCommand` - a derivative of
ReactiveCommand that encapsulates the common pattern of "Fire
asynchronous command, then marshal result back onto dispatcher
thread". Allows you to set a maximum level of concurrency as well
(i.e. "I only want 3 inflight requests" - when the maximum is
reached, CanExecute returns false). `ReactiveObject` - a ViewModel
object based on Josh Smith's implementation, that also implements
IObservable as a way to notify property changes. It also allows a
straightforward way to observe the changes of a single property.
`ReactiveValidatedObject` - a derivative of ReactiveObject that is
validated via DataAnnotations by implementing IDataErrorInfo, so
properties can be annotated with their restrictions and the UI will
automatically reflect the errors.
`ObservableAsPropertyHelper<T>` - a class that easily lets you
convert an IObservable<T\> into a property that stores its latest
value, as well as fires NotifyPropertyChanged when the property
changes. This is really useful for combining existing properties
together and replacing IValueConverters, since your ViewModels will
also be IObservables. `StopwatchTestScheduler` - this class allows
you to enforce time limits on items scheduled on other threads. The
main use for this is in unit tests, as well as being able to say
things in Debug mode like, "If any item runs in the Dispatcher
scheduler for longer than 400ms that would've made the UI
unresponsive, crash the application".

Blend SDK Integration
---------------------

`AsyncCommandVisualStateBehavior` - this behavior will watch a
ReactiveAsyncCommand and transition its target to different states
based on the command's status - for example, displaying a Spinner
while a command is running. `FollowObservableStateBehavior` - this
behavior will use the output of an IObservable<string\> and call
VisualStateManager.GoToState on its target; using Observable.Merge
makes it fairly straightforward to build a state machine based on
the changes in the ViewModel. `ObservableTrigger` - this trigger
will fire when an IObservable calls OnNext and can be tied to any
arbitrary Expression Action.

Other stuff that's useful
-------------------------

`MemoizingMRUCache` - this class is non-threadsafe most recently
used cache, and can be used to cache the results of expensive
lookups. You provide thefunction to use to look up values that
aren't known, then it will save the results. It also allows a
"destructor" to be run when an item is released from the cache, so
you can use this to manage an on-disk file cache as well (where the
"Get" function downloads a file, then the "Release" function
deletes it). `QueuedAsyncMRUCache` - this class is by far the most
complicated in this library, its goals are similar to
MemoizingMRUCache, but instead of returning the result immediately,
it will schedule a Task to run in the background and return an
IObservable representing the result (a Future). Once the Future
completes, its result is cached so subsequent requests will come
from memory. The advantage of this class is that subsequent
identical requests will block on the outstanding one (so if you ask
for "foo.com" on 3 separate threads, one of them will send out the
web request and the other two threads will receive the result as
well). This class also allows you to place a blocking limit on the
number of outstanding requests, so that further requests will block
until some of the inflight requests have been satisfied.
`IEnableLogger` - this is an implementation of a simple logger that
combines some of log4net's syntax with the ubiquity of the Rails
logger - any class that implements the dummy IEnableLogger
interface will able to access a logger for that class (i.e.
`this.Log().Warn("Something bad happened!");`)



# ReactiveXaml series: ReactiveCommand


What is ReactiveCommand
-----------------------

`ReactiveCommand` is an ICommand implementation that is
simultaneously a RelayCommand implementation, as well as some extra
bits that are pretty motivating. Let's jump right into the first
example:

    // This works just like Josh Smith's RelayCommand
    var cmd = ReactiveCommand.Create(x => true, x => Console.WriteLine(x));
    cmd.CanExecute(null);
    >> true

    cmd.Execute("Hello");
    "Hello"


Well that's boring, where's the fun stuff??
-------------------------------------------

However, here's where it gets interesting - we can also provide
IObservable<bool\> as our CanExecute. For example, here's a command
that can only run when the mouse is up:

    var mouseIsUp = Observable.Merge(
        Observable.FromEvent<MouseButtonEventArgs>(window, "MouseDown")
                  .Select(_ => false),
        Observable.FromEvent<MouseButtonEventArgs>(window, "MouseUp")
                  .Select(_ => true),
    ).StartWith(true);

    var cmd = new ReactiveCommand(mouseIsUp);
    cmd.Subscribe(x => Console.WriteLine(x));


Or, how about a command that can only run if two other commands are
disabled:

    // Pretend these were already initialized to something more interesting
    var cmd1 = new ReactiveCommand();
    var cmd2 = new ReactiveCommand();

    var can_exec = cmd1.CanExecuteObservable
        .CombineLatest(cmd2.CanExecuteObservable, (lhs, rhs) => !(lhs && rhs));

    var new_cmd = new ReactiveCommand(can_exec, Console.WriteLine);


One thing that's important to notice here, is that the command's
CanExecute updates **immediately**, instead of relying on
CommandManager.RequerySuggested. If you've ever had the problem in
WPF or Silverlight where your buttons don't reenable themselves
until you switch focus or click them, you've seen this bug. Using
an IObservable means that the Commanding framework knows *exactly*
when the state changes, and doesn't need to requery every command
object on the page.

What about Execute?
-------------------

This is where ReactiveCommand's IObservable implementation comes in
- ReactiveCommand itself can be observed, and it provides new items
whenever Execute is called (the items being the parameter passed
into the Execute call). This means, that Subscribe can act the same
as the Execute Action, or we can actually get a fair bit more
clever. For example:


    var cmd = ReactiveCommand.Create(x => x is int, null);

    cmd.Where(x => ((int)x) % 2 == 0)
       .Subscribe(x => Console.WriteLine("Even numbers like {0} are cool!", x));

    cmd.Where(x => ((int)x) % 2 != 0)
       .Timestamps()
       .Subscribe(x => 
          Console.WriteLine("Odd numbers like {0} are even cooler, especially at {1}!", x.Value, x.Timestamp));

    cmd.Execute(2);
    >>> "Even numbers like 2 are cool!"

    cmd.Execute(5);
    >>> "Odd numbers like 5 are even cooler, especially at (the current time)!"


Sum it all up, like that guy in Scrubs does all the time
--------------------------------------------------------

Hopefully that gives you some motivating examples as to why
combining the Reactive Extensions with WPF is a really awesome
idea, and not just for drag and drop! In the rest of this series
I'll spend some time explaining the rest of the classes and their
use, as well as going through a small sample application that I've
written that I'll soon be posting.


# ReactiveXaml Series: ReactiveAsyncCommand

Motivation
----------

If you've done any WPF programming that does any sort of
interesting work, you know that one of the difficult things is that
if you do things in an event handler that take a lot of time, like
reading a large file or downloading something over a network, you
will quickly find that you have a problem: during the blocking
call, the UI turns black. Silverlight heads this off at the pass -
you can't even do blocking operations *at all*.

So you say, "Oh, I'll just run it on another thread!" *Then*, you
find the 2nd tricky part - WPF and Silverlight objects have
*thread affinity*. Meaning, that you can only access objects
**from the thread that created them**. So, at the end of the
computation when you go to run `textBox.Text = results;`, you
suddenly get an Exception.

Dispatcher.BeginInvoke solves this
----------------------------------

So, once you dig around on the Internet a bit, you find out the
pattern to solve this problem involves the Dispatcher:

`void SomeUIEvent(object o, EventArgs e) {     var some_data = this.SomePropertyICanOnlyGetOnTheUIThread;       var t = new Task(() => {         var result = doSomethingInTheBackground(some_data);           Dispatcher.BeginInvoke(new Action(() => {             this.UIPropertyThatWantsTheCalculation = result;         }));     }       t.Start(); }`
We use this pattern a lot, let's make it more succinct
------------------------------------------------------

So, the idea of `ReactiveAsyncCommand` is that *many times*, when
we run a command, we often are just:

1.  The command executes, we kick off a thread
2.  We calculate something that takes a long time
3.  We take the result, and set a property on the UI thread, using
    Dispatcher

Because we encapsulate the pattern, we can get other stuff for free
-------------------------------------------------------------------

ReactiveAsyncCommand attempts to capture that pattern, and makes
certain things easier. For example, you often only want
**one async instance running**, and the Command should be disabled
while we are still processing. Another common thing you would want
to do is, display some sort of UI while an async action is running
- something like a spinner control or a progress bar being
displayed.

Since ReactiveAsyncCommand derives from ReactiveCommand, it does
everything its base class does - you can use it identically, and
the Execute IObservable tells you when *workitems are queued.*.
What ReactiveAsyncCommand does that would be hard to do with
ReactiveCommand directly, is that it has code built-in to
**automatically keep track of in-flight workitems.**

The first pattern - running an Action in the background
-------------------------------------------------------

Here's a simple use of a Command, who will run a task in the
background, and only allow one at a time (i.e. its CanExecute will
return false until the action completes)

`var cmd = new ReactiveAsyncCommand(null, 1 /*at a time*/);  cmd.RegisterAsyncAction(i => {     Thread.Sleep((int)i * 1000); // Pretend to do work };  cmd.Execute(5 /*seconds*/); cmd.CanExecute(5);  // False! We're still chewing on the first one.`
Putting it all together
-----------------------

Remember, ReactiveXaml is a MVVM framework - to fully demonstrate
calculating a value and displaying it in the UI, it's easiest to
make a very simple MVVM application. Here's the XAML, and the basic
class:

`<window x:Class="RxBlogTest.MainWindow"         x:Name="Window" Height="350" Width="525">          <grid DataContext="{Binding ViewModel, ElementName=Window}">         <stackpanel HorizontalAlignment="Center"  VerticalAlignment="Center">             <textblock Text="{Binding DataFromTheInternet}" FontSize="18"/>              <button Content="Click me!" Command="{Binding GetDataFromTheInternet}"                      CommandParameter="5" MinWidth="75" Margin="0,6,0,0"/>         </stackpanel>     </grid> </window>`
And the codebehind:

`using System; using System.Threading; using System.Windows; using ReactiveXaml;  namespace RxBlogTest {     public partial class MainWindow : Window     {         public AppViewModel ViewModel { get; protected set; }         public MainWindow()         {             ViewModel = new AppViewModel();             InitializeComponent();         }     }      public class AppViewModel : ReactiveValidatedObject     {         ObservableAsPropertyHelper[string] _DataFromTheInternet;         public string DataFromTheInternet {             get { return _DataFromTheInternet.Value; }         }          public ReactiveAsyncCommand GetDataFromTheInternet { get; protected set; }     } }`
This is a simple MVVM application, whose ViewModel has two items -
a Command called "GetDataFromTheInternet", and a place to store the
results, a property called "DataFromTheInternet". I'll describe
ObservableAsPropertyHelper later, but you can think of it as a
class that "Remembers the latest value of an IObservable".

Using ReactiveAsyncCommand
--------------------------

The difference between RegisterAsyncAction and
RegisterAsyncFunction is the *return value*. The latter function
returns an IObservable representing the results that
*will be returned*. For async calls, you can often think of
IObservable as a **Future Result**, that is, a "promise" of a
result (or an Exception if something goes pear-shaped). In this
case, our IObservable represents the "output" pipeline, and will
produce results every time someone fires the command, one per
Execute().

Here's how we actually implement the async function, in the
ViewModel constructor.

`public AppViewModel() {     GetDataFromTheInternet = new ReactiveAsyncCommand(null, 1 /*at a time*/);      //     // This function will return a "stream" of results, one per invocation     // of the Command     //      var future_data = GetDataFromTheInternet.RegisterAsyncFunction(i => {         Thread.Sleep(5 * 1000); // This is a pretend async query         return String.Format("The Future will be {0}x as awesome!", i);     });      // OAPH will "watch" future_data, and raise property changes when new values     // come in. It'll also provide the latest result that came in.     _DataFromTheInternet = new ObservableAsPropertyHelper<string>(future_data,         x => RaisePropertyChanged("DataFromTheInternet")); } </string>`
Why is this cool?
-----------------

Notice what I *didn't* have to do here: I didn't have to use any
sort of explicit async mechanism like a Task or a new Thread, I
didn't have to marshal data back to the UI thread using
Dispatcher.BeginInvoke, and my code reads way more like a simple,
single-threaded application again, instead of chaining async
invocations. Stuff like this is why I'm really excited about some
of the concepts in ReactiveXaml.

Furthermore, there's something else here that's very motivating:
**testability**. Using Dispatcher.BeginInvoke means that we're
assuming that a Dispatcher exists and works. If you're in a unit
test runner, *this isn't true*. Which means, your Commanding code
if you're using other MVVM frameworks that don't handle this
*isn't testable.* ReactiveXaml automatically detects whether you
are in a test runner, and changes its default IScheduler to not use
the Dispatcher. Test code still works, without hacking your
ViewModel code at all.

Where's the Code?
-----------------

Get the code here:
[ReactiveAsyncCmd.zip](http://paulbetts.org/blog_samples/ReactiveAsyncCmd.zip).
Also, I'm too lazy to correct the typo in the namespace, bt it
doesn't matter. Thoughts? Comments? Ideas?

ReactiveXaml series: A Sample MVVM application
A Sample App makes understanding ReactiveXaml way easier
--------------------------------------------------------

I used this application to help guide the API design around
ReactiveXaml, and I think it's a good illustration on how to use
this library to write WPF applications. The original goal of it was
to teach WPF to folks, so it has a *ton* of documentation in the
comments - it almost is "blogging via code", and I think it's a
really interesting way to express ideas, calling back somewhat to
Knuth's ideas of
[Literate Programming](http://en.wikipedia.org/wiki/Literate_Programming)

![image](http://blog.paulbetts.org/wp-photos/RxSampleApp1.png)  
*Our sample app helps us make a list of Awesome People*

Make sure to read the code!
---------------------------

A lot of people who originally saw this sample ran it, saw that it
didn't really do anything interesting, then forgot about it. The
app itself is *boring*, **the code is what's important!** Read
through it, there are lots of "essays" scattered throughout the
source code, they all start with the tag "COOLSTUFF" - searching
for it will help you find guidance on Rx, MVVM, as well as things
like layout and bindings.

![image](http://blog.paulbetts.org/wp-photos/RxSampleApp2.png)  
*This dialog shows validation, async web requests, and a clever use of Flickr*

Where's the code again?
-----------------------

It's at the
[ReactiveXaml repository on Github](http://github.com/xpaulbettsx/ReactiveXaml)
- if any of you want to check this out but have trouble getting Git
to work, let me know via Email at
[paul@paulbetts.org](mailto:paul@paulbetts.org) or
[Email the mailing list](mailto:reactivexaml@googlegroups.com) and
I'll help you out.

ReactiveXaml series: ReactiveObject, and why Rx is awesome
ViewModels via ReactiveObject
-----------------------------

Like any other MVVM framework,
[ReactiveXaml](http://github.com/xpaulbettsx/ReactiveXaml) has an
object designed as a ViewModel class. This object is based on Josh
Smith's ObservableObject implementation in
[MVVM Foundation](http://mvvmfoundation.codeplex.com/) (actually,
many of the classes' inspiration come from MVVM Foundation, Josh
does awesome work!). The Reactive version as you can imagine,
implements INotifyPropertyChanged as well as IObservable so that
you can subscribe to object property changes.

Other things that are nice to have
----------------------------------

ReactiveObject also does a few nice things for you: first, when you
compile ReactiveXaml in Debug mode, it will print debug messages
using its logging framework whenever a property changes. Another
example is, implementing the standard pattern of a property that
raises the changed event is a few lines shorter:
`int _someProp; public int SomeProp {     get { return _someProp; }     set { this.RaiseAndSetIfChanged(x => x.SomeProp, value);} }`

Compared to the traditional implementation which is a few lines
longer:
`int _someProp; public int SomeProp {     get { return _someProp; }     set {         if (_someProp == value)             return;         _someProp = value;         RaisePropertyChanged("SomeProp");     } }`

Some philosophy
---------------

A lot of examples of the Reactive Extensions make its domain appear
really constrained, like the only thing it's ever useful for is
either handing web service requests and implementing drag-and-drop.
However, here's the key thing to realize -
**a property change notification is an event.** Once you realize
that Reactive Programming applies to any time an object changes
state, Rx suddenly becomes far more applicable to any programming
domain - really, Rx is a library to model state machines that are
context-free with respect to threading.

Abstracting away context is critical for a multicore + cloud world
------------------------------------------------------------------

What do I mean by "context-free"? Well, just as LINQ abstracts away
the idea of a for loop into these operations like "Select" and
"Where", where the *implementation* is separate from the
*semantics*, Rx does the same thing. When you call Where() on an
IEnumerable, the default implementation is a simple loop - but add
an .AsParallel(), and suddenly the same code is now running on
multiple cores. Use Dryad, and the same code is running on multiple
machines in an HPC cluster.

In traditional imperative programming, we are always **explicit**
about the thread context in which we were running in - to put work
on another thread, we had to call `new Thread()`. The really
interesting thing about Rx, and the reason that Rx is made by the
"Cloud Programmability Group" inside MS, is that Rx is *also* an
abstraction above context - in Rx, you usually *don't care* what
thread you're running on, and you only have to specify it when you
*explicitly do care* via ObserveOn (like to deal with WPF's thread
affinity). Observables are essentially a way to declaratively write
dependencies between incoming data sources without explicitly
specifying the **mechanism** of their synchronization, only their
**semantics**.

Do you see where I'm going with this? If the concept of "Lock" and
"Thread" are no longer concretely tied to a kernel thread and a
Critical Section, this means that you can write the same Rx code,
and go from a single thread, event-based model with no locks, to a
multicore model that synchronizes via locking, to a cluster of
computers in a lab which synchronize via a message-passing model,
to a giant cloud computing array that synchronizes via a service
bus. **I think that's awesome.**

ReactiveXaml series: Implementing search with
ObservableAsPropertyHelper
Implementing an auto-search TextBox using Rx and ReactiveXaml
-------------------------------------------------------------

One of the most important classes in
[ReactiveXaml](http://github.com/xpaulbettsx/ReactiveXaml) called
`ObservableAsPropertyHelper` is a class that allows you to take an
IObservable and convert it into a read-only, change notifying
property. This class is really useful for exposing the results of
your code (i.e. the "output"). This class makes it easy to complete
the scenario that was described in the
[previous blog post about ReactiveAsyncCommand](http://blog.paulbetts.org/index.php/2010/06/27/reactivexaml-series-reactiveasynccommand/).
One of the cool things about ObservableAsPropertyHelper, is that it
guarantees that it will run notifications on the UI thread via the
Dispatcher so that you don't have to think about what thread the
observable notification came in on.

The sample app
--------------

[![image](http://blog.paulbetts.org/wp-photos/OAPHSample.png)](http://www.paulbetts.org/blog_samples/OAPHSample.zip)  
*Click on the image to download the sample project.*

Going through the code
----------------------

First, let's look at our main data item - a Flickr search result
item. Since we will never change these objects, we don't need any
INotifyPropertyChanged goo, just regular old auto-properties:
`public class FlickrPhoto {     public string Title { get; set; }     public string Description { get; set; }     public string Url { get; set; } }`

Now, the app data model - there's two real bits; the
**current search text**, and the **List of FlickrPhoto results**.
In ReactiveXaml, all of this code below is boilerplate - these code
chunks are just some stuff to memorize or put into a snippet and
never look at it again.
*Note: once again because of a syntax highlighting glitch, generics are using [] instead of < \>*
`public class AppViewModel : ReactiveValidatedObject {     //     // This is the canonical way to make a read-write property     //      string _SearchTerm;     public string SearchTerm {         get { return _SearchTerm; }         set { this.RaiseAndSetIfChanged(x => x.SearchTerm, value); }     }      //     // This is the canonical way to make a read-only property whose value     // is backed by an IObservable     //      ObservableAsPropertyHelper[List[FlickrPhoto]] _Photos;     public List[FlickrPhoto] Photos {         get { return _Photos.Value; }     }      ObservableAsPropertyHelper[Visibility] _SpinnerVisibility;     public Visibility SpinnerVisibility {         get { return _SpinnerVisibility.Value; }     }      public ReactiveAsyncCommand ExecuteSearch { get; protected set; } }`

Now here's the interesting part
-------------------------------

Our goal is to write a search box which automatically issues
searches in the background as the user types, similar to what most
browsers do with the address bar. However, there are a number of
tricky aspects to this:

-   We don't want to issue too many requests, especially when the
    user is still typing, so wiring something directly to KeyUp would
    be lousy.
-   Don't issue queries for empty strings, and don't issue the same
    query 2x (for example, if the user types "foo", then quickly hits
    Backspace, then retypes 'o', we should realize that we already have
    the right results)
-   The delay should be consistent, so having a global timer won't
    work because sometimes the user will hit the key right before the
    timer fires, so the delay will vary wildly between the max time and
    instantaneous.

Implementing this properly using traditional methods would be
absolutely awful. Here's the code on how we do it, and it's 5 lines
in the constructor:
`public AppViewModel() {     ExecuteSearch = new ReactiveAsyncCommand(null, 0);      //     // Take the inflight items and toggle the visibility     //      var should_spin = ExecuteSearch.ItemsInflight.Select(x => x > 0 ? Visibility.Visible : Visibility.Collapsed);      //     // This was described last time too, we actually do the async function     // here and RegisterAsyncFunction will return an IObservable which     // gives us the output, one item per invocation of ExecuteSearch.Execute     //       var results = ExecuteSearch.RegisterAsyncFunction(         term => GetSearchResultsFromFlickr((string)term));      //     // Here's the awesome bit - every time the SearchTerm changes     // throttled to every 800ms (i.e. drop changes that are happening     // too quickly). Grab the actual text, then only notify on unique     // changes (i.e. ignore "A" => "A"). Finally, only tell us when      // the string isn't empty. When *all* of those things are true,     // fire ExecuteSearch and pass it the term.     //       this.ObservableForProperty[AppViewModel, string]("SearchTerm")         .Throttle(TimeSpan.FromMilliseconds(800))         .Select(x => x.Value).DistinctUntilChanged()         .Where(x => !String.IsNullOrWhiteSpace(x))         .Subscribe(ExecuteSearch.Execute);      //     // This code is also boilerplate, it's the standard way to take our     // observable and wire it up to the property, giving it an initial     // value.     //      _SpinnerVisibility = new ObservableAsPropertyHelper[Visibility](         should_spin, x => RaisePropertyChanged("SpinnerVisibility"), Visibility.Collapsed);      _Photos = new ObservableAsPropertyHelper[List[FlickrPhoto]](         results, _ => RaisePropertyChanged("Photos")); }`

Here's the code that actually does the work as an aside, it's not
nearly as pretty:
`// // If you don't understand this code, don't worry about it, I just got lazy. // We're just hack-parsing the RSS feed and grabbing out title/desc/url and // newing up the list of FlickrPhotos while blatantly abusing Zip. //  public static List[FlickrPhoto] GetSearchResultsFromFlickr(string search_term) {     var doc = XDocument.Load(String.Format(CultureInfo.InvariantCulture,         "http://api.flickr.com/services/feeds/photos_public.gne?tags={0}&format=rss_200",         HttpUtility.UrlEncode(search_term)));     if (doc.Root == null)         return null;      var titles = doc.Root.Descendants("{http://search.yahoo.com/mrss/}title")         .Select(x => x.Value);     var descriptions = doc.Root.Descendants("{http://search.yahoo.com/mrss/}description")         .Select(x => HttpUtility.HtmlDecode(x.Value));     var items = titles.Zip(descriptions,         (t, d) => new FlickrPhoto() { Title = t, Description = d }).ToArray();      var urls = doc.Root.Descendants("{http://search.yahoo.com/mrss/}thumbnail")         .Select(x => x.Attributes("url").First().Value);      var ret = items.Zip(urls, (item, url) => { item.Url = url; return item; }).ToList();     return ret; }`

ReactiveXaml series: Using MemoizingMRUCache
Memoization and Caching
-----------------------

One thing that is useful in any kind of programming is having a
look-up table so that you don't have to spend expensive calls to
fetch the same data that you just had recently, since fetching the
data and passing it around via parameters often gets ugly. A better
way is to use a cache - store values we've fetched recently and
reuse them. So, a naïve approach would be to store the data off in
a simple Dictionary. This might work for awhile, but you soon
realize as Raymond Chen says, "Every cache has a cache policy,
whether you know it or not." In the case of a Dictionary, the
policy is unbounded - an unbounded cache is a synonym for 'memory
leak'.

To this end, one of the things that comes with ReactiveXaml is a
class called `MemoizingMRUCache`. As its name implies, it is a
*most recently used* cache - we'll throw away items whose keys
haven't been requested in awhile; we'll keep a fixed limit of items
in the cache, unlike other approaches involving WeakReference that
keep references to items only if they're used on some other thread
at the time. Since most desktop / Silverlight applications aren't
so massively multithreaded as a web application, using a
WeakReference approach means we'll just get constant cache misses.

Using MemoizingMRUCache
-----------------------

Really when it comes down to it, you can just think of
MemoizingMRUCache as just a proxy for a function - when you call
`Get`, it's going to invoke the function you provided in the
constructor. One thing that's important to understand with this
class, is that your function must be a function
*in the mathematical sense* - i.e. the return value for a given
parameter **must** always be identical. Another thing to remember
is that this class **is not implicitly thread-safe** - unlike
QueuedAsyncMRUCache, if you use it from multiple threads, you have
to protect it via a lock just like a Dictionary or a List.

Here's a motivating sample:

`// Here, we're giving it our "calculate" function - the 'ctx' variable is  // just an optional parameter that you can pass on a call to Get. var cache = new MemoizingMRUCache[int, int]((x, ctx) => {     Thread.Sleep(5*1000);     // Pretend this calculation isn't cheap     return x * 100; }, 20 /*items to remember*/);  // First invocation, it'll take 5 seconds cache.Get(10); >>> 1000   // This returns instantly cache.Get(10); >>> 1000   // This takes 5 seconds too cache.Get(15); >>> 1500`
Maintaining an on-disk cache
----------------------------

MemoizingMRUCache also has a feature that comes in handy in certain
scenarios: when a memoized value is evicted from the cache because
it hasn't been used in awhile, you can have a function be executed
with that value. This means that MemoizingMRUCache can be used to
maintain on-disk caches - your Key could be a website URL, and the
Value will be a path to the temporary file. Your OnRelease function
will delete the file on the disk since it's no longer in-use.

Some other useful functions
---------------------------



-   **TryGet** - Attempt to fetch a value from the cache only
-   **Invalidate** - Forget a cached key if we've remembered it and
    call its release function
-   **InvalidateAll** - Forget all the cached keys and start from
    scratch

ReactiveXaml Series: On combining notifications
I was excited today when I got my first Email post
[to the Google Group for RxXaml](http://groups.google.com/group/reactivexaml/),
and even better, it was a **great** question; what the poster was
asking about really strikes to the core of why Rx and ReactiveXaml
are compelling in my mind. In my experimentation, I've come across
a number of useful patterns that I should've mentioned earlier - I
showed you how to *get* the notifications, but not how to use
them!

When I say 'notification', you have to read into this term very
broadly and kind of stretch your brain a bit: as a reminder, here
are some examples of what are notifications in Rx and RxXaml:

-   A simple .NET event (i.e. via Observable.FromEvent)
-   Whenever a property changes (via ReactiveObject)
-   When an ICommand is invoked (ReactiveCommand)
-   Any time any sort of asynchronous operation completes (via
    Observable.FromAsyncCommand, ReactiveAsyncCommand, or
    QueuedAsyncMRUCache)
-   In response to an explicit notification (via a Subject)

Combining notifications in meaningful ways
------------------------------------------

One of the most powerful parts of the Reactive Extensions is its
ability to combine single events compositionally - when I describe
what Rx is to people, I often use the description, "Rx gives you
the ability to take simple events and combine them together into
something more specific and useful - I don't *really* care when the
'MouseUp' and 'KeyDown' events happen, I want to know when the
'User dropped a file on the top left corner' happens - tell me
about *that*."

To this effect, there are several tricks that we can do. The first
one is, that you must remember that ReactiveObject fires its
IObservable when **any** property changes - this means, that it's
very easy to watch an entire object. Often, this is useful enough -
when it isn't, `Where`helps you out:

`ReactiveObject Toaster;  // Any change will print something Toaster.Subscribe(x => Console.WriteLine("{0} changed!", x.PropertyName);  // This is an observable that only notifies when the Foo property changes var FooChanged = Toaster.Where(x => x.PropertyName == "Foo");`
Merge, CombineLatest, and Zip - the 'And' and 'Or' of Rx
--------------------------------------------------------

So, to combine several IObservables, we have a few useful methods
that stand out. The first is `Observable.Merge`: as its name
implies, Merge takes several IObservables *of the same type*, and
returns an IObservable that fires when
**any one of its inputs fires**. Thinking in a boolean sense,
**Merge is kind of like *Or***. Having to be of the same type isn't
as onerous of a requirement:

`IObservable[float] O1; IObservable[int] O2; IObservable[string] O3;  // Tell me when *any* of these 3 send a notification var result = Observable.Merge(     O1.Select(_ => true), O2.Select(_ => true), O3.Select(_ => true) );`
One of the difficulties of Merge that can sometimes bite you, is
that it is **stateless** - when you get a notification about O1,
you don't have any knowledge about what items came in on O2 or O3.
For two IObservables, we have a handy method called
`Observable.CombineLatest`. This method will "remember" the last
item that came in on both sides - when O1 changes, it will give you
the new O1 *and the latest value of O2*. Furthermore, we can take
the **result** and expose it as a change-notifying property
[via ObservableAsPropertyHelper](http://blog.paulbetts.org/index.php/2010/07/05/reactivexaml-series-implementing-search-with-observableaspropertyhelper/).

`// Subjects are just IObservables that we can trigger by-hand // They're the mutable variables of Rx Subject[int] s1; Subject[int] s2;  // Combine s1 with s2 and write its output to Console s1.CombineLatest(s2, (a,b) => a * b).Subscribe(Console.WriteLine);  s1.OnNext(5);  // Nothing happens, no value for s2  s2.OnNext(10);  // 10 came in, combine the 10 with whatever s1 was (5) >>> 50  s2.OnNext(20); // 20 came in, still use s1's latest value >>> 100  s1.OnNext(2); // s1 is 1, take s2's latest value (20) >>> 40`
Finally, we have `Observable.Zip`. Like the other two, this
function also combines observables, but this function like its
IEnumerable counterpart, is only concerned about
**pairs of items**. This means, it's more like "And" than the other
two (remember that it's extremely unlikely that notifications will
come in at the *exact same time* so an "Observable.And" wouldn't
make much sense). Zip will not yield elements until it has **both**
of its "slots" filled for the next item.

`Subject[int] s1; Subject[int] s2;  s1.Zip(s2, (a,b) => a * b).Subscribe(Console.WriteLine);  s1.OnNext(2); // Nothing, no pair yet s1.OnNext(5); // Still no pair s2.OnNext(10); // We've got a pair (2,10), let's send it down >>> 20  s1.OnNext(10); // s2's empty, no pair s2.OnNext(1); // 5 * 1 >>> 5 s2.OnNext(10); // 10*10 >>> 100 s2.OnNext(100); // s1's empty, no output`
Combining Notifications for Visual State Manager
------------------------------------------------

Here's another clever trick that I really like - often, we need to
change the visual state on a variety of different notifications of
different types and are unrelated. Here's how to do it:

`IObservable[int] SomethingToWatch, SomethingElse; IObservable[float] AThirdThing; var state = Observable.Merge(     SomethingToWatch.Select(_ => "State1"),     SomethingElse.Select(_ => "State2"),     AThirdThing.Select(_ => "State3") );  state.Subscribe(x =>      VisualStateManager.GoToState(this, x, true));`
Observable.Merge can also be used along with Scan to keep a
reference count, check out
[this example from ReactiveAsyncCommand](https://github.com/xpaulbettsx/ReactiveUI/blob/master/ReactiveUI.Xaml/ReactiveAsyncCommand.cs#L83)
where we use two observables and Select them to 1 and -1, then keep
a running count via Scan.

ReactiveXaml Series: QueuedAsyncMRUCache - the async version of
MemoizingMRUCache
A thread-safe, asynchronous MemoizingMRUCache
---------------------------------------------

As we saw
[in a previous entry](http://blog.paulbetts.org/index.php/2010/07/13/reactivexaml-series-using-memoizingmrucache/),
MemoizingMRUCache is great for certain scenarios where we want to
cache results of expensive calculations, but one disadvantage is
that it is fundamentally a single-threaded data structure:
accessing it from multiple threads, or trying to cache the results
of several in-flight web requests at the same time would result in
corruption. QueuedAsyncMRUCache solves all of these issues, as well
as gives us a new method called `AsyncGet`, which returns an
IObservable. This IObservable will fire exactly once, when the
async command returns.

What places would I actually want to use this class? Here's a
motivating example: you're writing a Twitter client, and you need
to fetch the profile icon for each message - a naive foreach loop
would be really slow, and even if you happened to write it in an
asynchronous fashion, you would *still* end up fetching the same
image potentially many times!

Using IObservable as a Future
-----------------------------

One of the things that an IObservable encapsulates is the idea of a
[Future](http://en.wikipedia.org/wiki/Future_(programming)),
described simply as the future result of an asynchronous operation.
The pattern is implemented via an IObservable that only produces
one element then completes. Using IObservable as a Future provides
a few handy things:

-   IObservables let us block on the result if we want, via
    Observable.First().
-   IObservables have built-in error handling via OnError, so we
    can also handle the case where something goes pear-shaped.
-   We can easily group several IObservables together via
    Observable.Merge and wait for any (or all) of them.

A difficult problem - preventing concurrent identical requests
--------------------------------------------------------------

Furthermore, QueuedAsyncMRUCache solves a tricky problem as well:
let's revisit the previous example. As we walk the list of
messages, we will asynchronously issue WebRequests. Imagine a
message list where every message is from the same user:

For the first item, we'll issue the WebRequest since the cache is
empty. Then, we'll go to the 2nd item - since the first request
probably hasn't completed, we'll *issue the same request again*. If
you had 50 messages and the main thread was fast enough, you could
end up with 50 WebRequests **for the same file!**

What *should* happen? When the 2nd call to AsyncGet occurs, we need
to check the cache, but we also need to check the list of
outstanding requests. Really, for every possible input, you can
think of it being in one of three states: either in-cache,
in-flight, or brand new. QueuedAsyncMRUCache ensures (through a lot
of code!) that all three cases are handled correctly, in a
thread-safe manner.

Making INotifyPropertyChanged type-safe using Expressions
[Some folks on the mailing list](http://groups.google.com/group/reactivexaml/browse_thread/thread/c6fc10f65de05be0)
for [ReactiveXaml](http://bit.ly/rxxaml) rightly pointed out some
code in ReactiveXaml that could really use some work - my
implementation of INotifyPropertyChanged's "RaiseAndSetIfChanged".
For the impatient,
[here's how I did it](http://github.com/xpaulbettsx/ReactiveXaml/commit/98001677500eeed55b53b63642a2f38292987a4d#L3R143)

Trying to make RaisePropertyChanged less verbose
------------------------------------------------

This method is a helper method designed to encapsulate the
following common code pattern when implementing a property:

1.  Make sure that the property is different than the original
    value
2.  Set the property to the new value
3.  Call *RaisePropertyChanged* with the name of the property that
    has changed

Doing this via a helper is a little bit tricky - the naive approach
often forgets \#1 entirely, or will try to do \#2 and \#3 in the
opposite order, resulting in WPF/Silverlight not correctly
updating. Another problem is that we usually have to pass in a
string literal to RaisePropertyChanged (as well as
RaiseAndSetIfChanged). While I'm a Rubyist and string literals
don't bother *me*, most .NET developers are annoyed by it, and it
also can lead to really evil-to-debug issues after refactoring.

Using Expression Trees to implement a better version
----------------------------------------------------

Here's what the new syntax for defining properties looks like:

`string _SomeProperty;  public string SomeProperty {      get { return _SomeProperty; }      set { _SomeProperty = this.RaiseAndSetIfChanged(x => x.SomeProperty, value); }  }`
Much improved, and fairly clean (we could do way better if C\# had
Macros, but I digress). Compare this with the old, dumb looking
syntax:

`string _SomeProperty;  public string SomeProperty {      get { return _SomeProperty; }      set { _SomeProperty = this.RaiseAndSetIfChanged(_SomeProperty, value,          x => _SomeProperty, "SomeProperty");     } }`
Or doing this by-hand:

`string _SomeProperty;  public string SomeProperty {      get { return _SomeProperty; }      set {          if (_SomeProperty == value)             return;          _SomeProperty = value;         RaisePropertyChanged("SomeProperty");     } }`
How can we do this? Expression Trees! When we write
*Expression<Func<T\>\>* as a type instead of *Func<T\>*, instead of
getting a compiled method pointer, we get an object representing
the [AST](http://en.wikipedia.org/wiki/Abstract_syntax_tree) of the
lambda function. We'll use this to grep out the property name.

Astute readers will notice an issue - how can we correctly decide
*T* so that Intellisense works correctly? If we naïvely choose
'ReactiveObject' as T, none of our properties will be defined,
since they're all really defined on the derived class, not on
ReactiveObject itself. To solve this, we'll use a
[Mixin](http://en.wikipedia.org/wiki/Mixin) plus a cleverly typed
generic function - since the first parameter of an extension method
is the type itself, the compiler will automatically bolt this on to
any ReactiveObject while still treating it as the correct derived
type:

`public static class ReactiveObjectExpressionMixin {     public static TRet RaiseAndSetIfChanged[TObj, TRet](this TObj This,              Expression[Func[TObj, TRet]] Property,              TRet Value)         where TObj : ReactiveObject     {     } }`
Some tricky caveats
-------------------

It's not all biscuits and gravy though, there are a few subtle
implementation details that might catch you when using this -
remember though, that you can always fall back to writing it
by-hand via RaisePropertyChanged, this method is solely a helper:

1.  You **must** name the field that backs your property as
    `_TheFieldName` - we use Reflection to set the backing field
2.  Writing the 'this' in 'this.RaiseAndSetIfChanged' isn't
    optional - otherwise the extension method won't be invoked, the old
    busted version will

Calling Web Services in Silverlight using ReactiveXaml
One of the scenarios that our summer intern
[Roberto Sonnino](http://virtualdreams.com.br/blog/) pointed out
over the summer that was somewhat annoying in
[ReactiveXaml](http://bit.ly/reactivexaml) (my MVVM library that
integrates the Reactive Extensions for .NET), was making RxXaml
more friendly to asynchronous web service calls in Silverlight.
Since this is a pretty important scenario, I decided to hack on it
some (it's always been possible, just not as easy).

Getting our function prototype correct
--------------------------------------

Remember from
[an earlier post](http://blog.paulbetts.org/index.php/2010/08/23/reactivexaml-series-queuedasyncmrucache-the-async-version-of-memoizingmrucache/)
that an IObservable can be used as a Future, a "box" that will
eventually contain the result of a web service call or other
asynchronous function, or the error information. So to this end,
we'd really like our web service calls to all be vaguely of the
form:

`IObservable[Something] CoolWebServiceCall(object Param1, object Param2 /*etc*/);`
However, we know that neither HttpWebRequest nor the web service
generated code looks *anything* like that. Normal Silverlight
objects use an OnCompleted event to fire a callback once the call
completes. The Rx framework designers saw this coming however, and
gave us a pretty handy function to deal with it:
`Observable.FromAsyncPattern()`. This function will take a
BeginXXXX/EndXXXX pair that follows the
[.NET Asynchronous Pattern](http://msdn.microsoft.com/en-us/library/ms228963.aspx)
and in return, will give us a Func that follows the form above,
without us having to write some boilerplate code to connect the
async method to an IObservable.

An example: looking at the Bing Translation API
-----------------------------------------------

One of the most simple public web services is the
[Bing Translation API](http://msdn.microsoft.com/en-us/library/ff512435.aspx),
so its Translate() method is a good candidate for our example. The
synchronous signature is:

`string Client.Translate(string appId, string text, string fromLang, string toLang);`
Here's how we could take this and turn it into an Rx-friendly Async
function - don't be scared off by the five template parameters,
they're just the types of the parameters and return value, in the
same order as a Func<T\>:

`var client = new LanguageServiceClient(); var translate_func = Observable.FromAsyncPattern[string,string,string,string,string]     (client.BeginTranslate, client.EndTranslate);  IObservable[string] future = translate_func(appId, "Hello World!", "en", "de"); string result = future.First();   // This will *wait* until the call returns!! >>> "Guten Tag, Welt!"  // // Let's try with an array... //  var input = new[] {"one", "two", "three"};  // Fire off three asynchronous web service calls at the same time var future_items = input.ToObservable()     .SelectMany(x => translate_func(appId, x, "en", "fr"));  // This waits for *all* the web service calls to return string[] result_array = future_items.ToArray();   >>> ["un", "deux", "trois"]`
An important note for Silverlight!
----------------------------------

Silverlight's web service generated client code does something a
bit annoying - it **hides away** the BeginXXXX/EndXXXX calls,
presumably to make the Intellisense cleaner. However, they're not
gone, the way you can get them back is by casting the
MyCoolServiceClient object **to its underlying interface** (i.e.
the LanguageServiceClient object has a generated
ILanguageServiceClient interface that it implements)

Turning this into a Command
---------------------------

Once we've got the function, turning it into a command is easy via
[a new method](http://github.com/xpaulbettsx/ReactiveXaml/commit/1c22a7df2fec043e51b417ef6802541420f319a6)
introduced to
[ReactiveAsyncCommand](http://blog.paulbetts.org/index.php/2010/06/27/reactivexaml-series-reactiveasynccommand/)
- `RegisterObservableAsyncFunction`. This method is almost
identical to RegisterAsyncFunction, but instead of expecting a
synchronous Func which will be run on the
[TPL Task pool](http://msdn.microsoft.com/en-us/library/dd460717.aspx),
it expects a Func that returns an IObservable as described above.
Here's a simple example of a good ViewModel object that
demonstrates this:

`public class TranslateViewModel : ReactiveObject {     //     // Input text     //       string _TextToTranslate;     public string TextToTranslate {         get { return _TextToTranslate; }         set { RaiseAndSetIfChanged(x => x.TextToTranslate, value); }     }      //     // The "output" property we bind to in the UI     //      ObservableAsPropertyHelper[string] _TranslatedText;     public string TranslatedText {         get { return _TranslatedText.Value; }     }      public ReactiveAsyncCommand DoTranslate { get; protected set; }      const string appId = "Get your own, buddy!";     public TranslateViewModel()     {         var client = new LanguageServiceClient();         var translate_func = Observable.FromAsyncPattern[string,string,string,string,string](                 client.BeginTranslate, client.EndTranslate);          // Only one web call at a time please!         DoTranslate = new ReactiveAsyncCommand(null, 1);          //         // 'x' is the CommandParameter passed in, which we will use as the         // source text         //          var results = DoTranslate.RegisterObservableAsyncFunction(             x => translate_func(appId, (string)x, "en", "de"));                      _TranslatedText = this.ObservableToProperty(             results, x => x.TranslatedText);     } }`
What does that get us?
----------------------

Let's review what this fairly short, readable code gets us - using
a few simple bindings, we'll have a fully non-blocking, responsive
UI that correctly handles a lot of the edge cases associated with
background operations: greying out the Button attached to the
Command while the web call is running, saving off the results, then
notifying the UI that there is something new to display so that it
updates instantly, without any tricky callbacks or mutable state
variables that have to be guarded by Lock statements to ensure
multithreaded safety. That's pretty cool.

ReactiveXaml Series: Displaying a 'Loading...' value
Some folks from the
[nRoute Framework](http://nroute.codeplex.com/Thread/View.aspx?ThreadId=227771)
were looking for an elegant way to handle displaying an
intermediate value during async updates - something like changing
the text to say "Working..." until the task is complete, then
displaying the results. I realized this was pretty easy to do with
[ReactiveXaml](http://bit.ly/rxxaml), so I wrote up some code -
here it is:

`public class CoolViewModel : ReactiveObject {     //     // Standard way in RxXaml to declare a notification-enabled property     //       string _InputData;     public string InputData {         get { return _InputData; }         set { this.RaiseAndSetIfChanged(x => x.InputData, value); }     }      // Our ICommand - invoking this will *begin* the async operation     ReactiveAsyncCommand DoubleTheString;      //     // OAPH will create an 'output' property - that is, a property who will      // be updated via an IObservable     //       ObservableAsPropertyHelper[string] _OutputData;     public string OutputData {         get { return _OutputData.Value; }     }      public CoolViewModel(Window MainWindow)     {         DoubleTheString = new ReactiveAsyncCommand(null, 1/*at a time*/);          IObservable[string] doubled_strings = DoubleTheString.RegisterAsyncFunc(x => {             // Pretend to be a slow function             Thread.Sleep(1000);             return String.Format("{0}{0}", x);         });          //         // ReactiveAsyncCommand will fire its OnNext when the command is *invoked*,         // and doubled_strings will fire when the command *completes* - let's use         // this to our advantage:         //          IObservable[string] result_or_loading = Observable.Merge(             DoubleTheString.Select(x => "Loading..."),             doubled_strings         );          // Hook up our new 'result_or_loading' to the output         _OutputData = this.ObservableToProperty(result_or_loading, x => x.OutputData);     } }`
Some annoying bugs to be aware of in ReactiveXaml for Silverlight
Despite [ReactiveXaml](http://github.com/xpaulbettsx/ReactiveXaml)
being available for WPF, Silverlight, and Windows Phone 7, I have a
confession to make: the vast majority of my use with this library
is in WPF. The test suite runs under WPF, all of the applications
that I am building to prototype RxXaml features are WPF
applications, etc. This isn't any slight against Silverlight, it
just happens to be what I'm usually working on. So, I assumed
naïvely that ReactiveXaml under SL4 would "just work".

It works now, after some fixups
-------------------------------

After some folks messaged me about some issues in the Silverlight
version, I decided to wire up the SL test runner to see if I could
reproduce some of these issues. Using a great test runner called
[StatLight](http://statlight.codeplex.com/), I could quickly see
that there are some bugs. Luckily, we now run the SL unit test
runner as part of the test suite, so hopefully this won't be the
case any more.

List of bugs fixed as of today
------------------------------

-   ReactiveCollection failed any time someone called "coll[x] =
    foo"; this is because Silverlight's ObservableCollection returns
    different results than the desktop CLR on
    NotifyCollectionChangedAction.Replace
-   Any method involving Expression Trees failed, because SL's
    Type.GetField doesn't appear to return any results if you pass
    BindingFlags to it
-   Notifications that were supposed to come in on the UI thread
    were coming in on other threads - this was a dumb typo on my part

Bugs that still need fixed
--------------------------

-   To actually get the unit test runner to work, you have to rig
    [this line](http://github.com/xpaulbettsx/ReactiveXaml/blob/master/ReactiveXaml/Interfaces.cs#L193)
    to say "return true" - I haven't yet figured out a way to determine
    whether we are in a unit test runner in Silverlight, since
    AppDomain.GetAssemblies() doesn't exist, though I've got a good
    idea I'll try soon.

An annoying caveat for SL4
--------------------------

On Silverlight 4, for security reasons, you cannot use
Type.GetField to access private types - whereas protection on the
desktop CLR is done at compile time, on SL4 this is enforced at
**run-time** as well. This really sucks for RxXaml, since it uses
reflection to set backing fields. The workaround is ugly, you have
to mark your backing field as public, or use the simpler
RaisePropertyChanged and write properties by-hand. So, here's the
way for SL4 to correctly write a read-write property:
`[IgnoreDataMember] public string _SearchText; public string SearchText {     get { return _SearchText; }     set { this.RaiseAndSetIfChanged(x => x.SearchText, value); } }`
Alternatively, if the 'public' really annoys you, here's how to do
it by-hand:
`string _SearchText; public string SearchText {     get { return _SearchText; }     set {         if (_SearchText == value)             return;          _SearchText = value;         this.RaisePropertyChanged("SearchText");     } }`

ReactiveXaml Series: Using ReactiveCollection to improve the Flickr
Search sample
One of the types
[in ReactiveXaml](http://blog.paulbetts.org/index.php/2010/10/30/reactivexaml-1-4-0-0-is-released-including-samples-and-binaries/)
that is quite useful that I haven't mentioned before on the blog is
an extension of WPF/SL's ObservableCollection (no relation to
IObservable), called (predictably enough), `ReactiveCollection`. It
works wherever you'd use an ObservableCollection in a traditional
M-V-VM library, but it has a few useful tricks worth mentioning.

The basics, plus watching item change notifications
---------------------------------------------------

As you might imagine, ReactiveCollection adds the following
IObservables that you can subscribe to:

-   BeforeItemsAdded / ItemsAdded
-   BeforeItemsRemoved / ItemsRemoved
-   CollectionCountChanging / CollectionCountChanged

In addition to this, ReactiveCollection can also watch item change
notifications on the items in the list, so you can express "Tell me
when the collection itself changes, *or any of its elements*". This
can be expensive for large lists, so it is disabled by default -
use the ChangeTrackingEnabled property to enable/disable it. As you
add/remove items, ReactiveCollection will be
subscribing/unsubscribing to the underlying objects.

Automatically creating ViewModel collections
--------------------------------------------

Since we now have information easily exposed about when items are
added/removed, it was fairly straightforward to create a Collection
who automatically follows another Collection via a Selector, called
"CreateDerivedCollection". Here's the most useful case:
`var Models = new ReactiveCollection[ModelClass](); var ViewModels = Models.CreateDerivedCollection(x => new ViewModelForModelClass(x));  // Now, adding / removing Models means we  // automatically have a corresponding ViewModel Models.Add(new Model("Hello!"));  ViewModels.Count(); >>> 1`

This really makes it easier follow the M-V-VM pattern properly when
it comes to collections - the code is editing the Models, and these
new Models are being projected into new ViewModels automatically.

Creating a collection from an Observable
----------------------------------------

A post-1.4 method called CreateCollection that I recently added
will also allow you to create a list based on an Observable,
optionally spacing each of the items apart by a certain amount of
time. If the time is specified, the list returned is always empty,
but slowly fills itself over time - this is great for populating
Listboxes in a more lifelike way. We'll see this in action in the
sample.

Improving our Flickr search sample
----------------------------------

In the
[sample for ObservableAsPropertyHelper](http://blog.paulbetts.org/index.php/2010/07/05/reactivexaml-series-implementing-search-with-observableaspropertyhelper/),
I showed how to implement a simple Flickr tag search. Using
CreateCollection, we can make the tiles appear on a delay which
looks snazzier.

[![image](http://blog.paulbetts.org/wp-photos/OAPHSample.png)  
](http://www.paulbetts.org/blog_samples/ReactiveCollectionSample.zip)
*Animations don't show up well as images. Click to download the sample project with binaries.*

Let's take a look at what changed - first, we changed our output
results from a List to a ReactiveCollection:
`ObservableAsPropertyHelper[ReactiveCollection[FlickrPhoto]] _Photos; public ReactiveCollection[FlickrPhoto] Photos {     get { return _Photos.Value; } }`

Next, we update GetSearchResultsFromFlickr to provide a
ReactiveCollection instead of a List, really only by changing the
first and last lines:
`public static ReactiveCollection[FlickrPhoto] GetSearchResultsFromFlickr(string search_term) {     /* [[[SNIP]]] This part is long and boring */      var ret = items.Zip(urls, (item, url) =>          { item.Url = url; return item; }).ToList();      // Take the return list, convert it to an Observable,      // then create a collection who will initially be     // empty, but we'll copy one item at a time every     // 250ms until we've copied everything from ret.     return ret.ToObservable().CreateCollection(TimeSpan.FromMilliseconds(250.0)); }`

Making it even snazzier - adding the fade-in
--------------------------------------------

Adding the items every 250ms is okay, but to really make this look
great we need an animation to fire whenever an item is added. To do
this, we override the ListBoxItemContainer template, and make these
changes (this has nothing to do with ReactiveXaml directly, but
it's a cool trick nevertheless):

1.  Make a copy of the ListBoxItemContainer template
2.  Set the Border opacity to zero (its initial state)
3.  Create a Storyboard which will animate the opacity back to 100%
    in 1.5 seconds (using the Power easing function)
4.  Add an Expression Blend EventTrigger on the Loaded event, that
    will kick off our Storyboard

Detecting whether your .NET library is running under a unit test
runner
Here's a piece of code that I found useful for
[ReactiveXaml](http://github.com/xpaulbettsx/ReactiveXaml), how to
detect whether you are running under the unit test runner. For
WPF/Silverlight testing, this is important to know since
Dispatcher.Current exists but is bogus (i.e. none of the things you
queue to it will run during the unit tests). The official version
comes
[from here](https://github.com/xpaulbettsx/ReactiveXaml/blob/master/ReactiveXaml/RxApp.cs#L86).

`public static bool InUnitTestRunner() {     string[] test_assemblies = new[] {         "CSUNIT",         "NUNIT",         "XUNIT",         "MBUNIT",         "TESTDRIVEN",         "QUALITYTOOLS.TIPS.UNITTEST.ADAPTER",         "QUALITYTOOLS.UNITTESTING.SILVERLIGHT",         "PEX",     };  #if SILVERLIGHT     return Deployment.Current.Parts.Any(x =>        test_assemblies.Any(name => x.Source.ToUpperInvariant().Contains(name))); #else     return AppDomain.CurrentDomain.GetAssemblies().Any(x =>       test_assemblies.Any(name => x.FullName.ToUpperInvariant().Contains(name))); #endif }`
Making Async I/O work for you, Reactive style
Earlier today, I read
[a fantastic article about the TPL](http://www.hanselman.com/blog/BackToParallelBasicsDontBlockYourThreadsMakeAsyncIOWorkForYou.aspx)
by Scott Hanselman. In it, he describes how to take a fairly
straightforward function to detect if a given Url responds, and
write it in an asynchronous fashion. As soon as I read it, I knew
that I had to write the
[Reactive Extensions for .NET](http://msdn.microsoft.com/en-us/devlabs/ee794896.aspx)
version!

How do the TPL and Rx.NET relate?
---------------------------------

Both of these technologies are intended to help make writing
asynchronous and concurrent programs easier and more
straightforward, so it's really easy to be confused as to which one
to use. You can often think of Task and IObservable for async calls
as the same thing - an object that represents a *future result*
that hasn't completed yet - we saw this in
[a previous blog post](http://blog.paulbetts.org/index.php/2010/09/26/calling-web-services-in-silverlight-using-reactivexaml/).
In an asynchronous function, we send out the request, but we don't
have the data - we have to return *something* that will allow us to
eventually get the result.

When it comes down to it, Task is really a *specialization* of
IObservable - Task is specifically designed to run on the TPL
threadpool, whereas IObservable abstracts away where the code will
execute unless you specify it explicitly.

Seeing the problem again
------------------------

Let's take a look at the synchronous version of the code again - we
want to take this and rewrite it so that it doesn't block:

Writing our initial stab at VerifyUrlAsync
------------------------------------------

Just like Scott's Task-based async function, we'll also define a
function that returns a future result. However, instead of using
Task as our return type, we'll define a function that returns
IObservable:

Now, let's see the implementation:

How can we use this?
--------------------

This method will not block, it will instantly return you an
IObservable representing the future result. So, there are a couple
of ways you can use the Observable to "unpack" the result:

Now, let's see how we can do arrays:
------------------------------------

The truly revolutionary thing about Rx.NET is how the same
primitive you used in LINQ now take on awesome new meanings when
applied to the domain of the future. So, the first thing we'll do
is take our array and convert it to an IObservable via
AsObservable. This means that the resulting IObservable will
produce *n* items, one for each element in the array, then
OnComplete.

The natural thing we would do to get the result is
`someObservable.Select(x => ValidateUrlAsync(x))`. However, there's
a problem - our type is now IObservable<IObservable<T\>\>; we now
have a "future list of futures" (thinking of IObservable as a
"future list" is a good analogy, whereas the web call is just a
"future list" with only one item). We need a way to flatten down
our future list back to IObservable<T\> - so what's the way to
flatten a list in LINQ? SelectMany, of course! SelectMany is the
secret behind writing async Rx code. Let's see how to do it:

The code above is still asynchronous - at no time will we block,
and it will instantly return. The SelectMany's default IScheduler
will run items on the TaskPool (actually in this case, we never
used any synchronous method so we will never block, even on a
Threadpool thread. To get the result, similar to the above method,
we'd have to call `First()` on it.

If we were to dump the IObservables at every point of the
computation, it'd look something like this:

`[ "http://foo", "http://bar" ] ===>   [ {"http://foo", false}, {"http://bar", false} ]  ===>  [ Dictionary ]`
Cool! Where can I learn more?
-----------------------------



-   The
    [Rx Hands-on-lab](http://blogs.msdn.com/b/rxteam/archive/2010/07/15/rx-hands-on-labs-published.aspx)
    is an awesome, thorough, and technically correct introduction to
    Rx.NET
-   The
    [Rx.NET forums](http://social.msdn.microsoft.com/Forums/en-US/rx/threads)
    are full of really smart, helpful people - I've learned a ton by
    reading through the forum posts
-   The
    [Rx.NET videos on Channel 9](http://channel9.msdn.com/Tags/rx-in-depth)
    are a great resource - the developers behind the library itself
    explain the concepts in an easy-to-understand way
-   My
    [blog series on ReactiveXaml](http://blog.paulbetts.org/index.php/category/programming/reactive-extensions/)
    and Rx.NET is also a good way to understand many practical uses of
    Rx, especially if you're writing desktop / Silverlight / WP7 apps.

Testing your ViewModels using Time Travel and ReactiveUI
Testing asynchronous ViewModel interactions is tough
----------------------------------------------------

When running under a unit test runner, ReactiveUI makes it fairly
straightforward to test interactions between commands and changing
properties. Fiddle with properties, execute the commands, Assert
what happens - done!

However, most non-trivial programs need to run something in the
background - talk to a web service, calculate something in the
background, etc. Testing this can be way more challenging, since it
is easy to deadlock yourself with the Immediate scheduler (the one
used by-default in a unit test) - when it comes down to it, there
is exactly **one** thread, and it can't be doing **two+** things at
a time. This will typically come into play when you use a blocking
call like *First()*, then find out your test runner never finishes.
In a non-Rx context, we typically try to test this via
Thread.Sleep() calls or Waits, which are really slow and often give
you really unpredictable results.

Using EventScheduler in a pinch
-------------------------------

What we need, is a replacement for RxApp.DeferredScheduler that is
actually *deferred*, to take the place of WPF/Silverlight's
Dispatcher. Enter EventLoopScheduler! We can use this to create a
"pretend" Dispatcher on-the-fly that we control:

This is alright, but it still will slow down our test suite by
quite a bit, waiting for network access. What's worse, if we were
testing something more complicated, we could get tests that pass
sometimes but not others, depending on the timing - this is a huge
time sink for QA folks who have to then debug the test failures.

Testing software via Time Travel?!
----------------------------------

The guys from DevLabs came up with a pretty ingenious way to solve
this. Let's look at the definition of IScheduler, the interface
through which we send all of our deferred processing:

So, we can schedule code to run right now, we can schedule it to
run after a certain amount of time has elapsed, and then there's
that third member: Now. You might ask, "Why do I need to know Now,
don't I get it from DateTime.Now?" Here's the clever bit:
*What if you made a scheduler where Now was settable?* This
scheduler would never run anything, just queue it to a list of
stuff to run. Then, when "Now" is set (i.e. we "move through
time"), we activate anything that *would have run* in that time
period.

How does the TestScheduler work?
--------------------------------

In fact, this is *exactly* how TestScheduler works. When Rx
operators call Schedule(), nothing happens. Then, TestScheduler has
two interesting methods, **Run()**, which will run all of the
queued items (i.e. execute anything that it can), and
**RunToMilliseconds()**, which lets you travel to a certain time
period *n* milliseconds away from *t=0*.

Faking out an asynchronous web call
-----------------------------------

Sounds great, right? Here's the caveat about TestScheduler though -
if you use any other asynchronous methods like Event or
Task.Wait(), it'll be tougher to integrate TestScheduler, since not
all sources of async'ness are going through the TestScheduler.
However, if you're using Rx in a project, I consider using other
sync/thread patterns to be an Rx Code Smell - like casting
IEnumerables in LINQ to Array because I happen to know it's an
Array.

Let's see how we can create a fake async Read method, that will
simulate taking up some time and returning a result:

The cool thing about this mock, is that if you used it in a normal
environment or under an EventLoopScheduler, it'd do exactly as it
said: wait 10 seconds, then return that array. Under the
TestScheduler, we'll make it return **immediately**!

Writing the Unit Test
---------------------

Here's how we could write the Unit Test above to execute instantly
using TestScheduler (actually fleshing out the MyCoolViewModel so
you can see how it's wired up). Inverting the control to actually
get the mock function here is pretty ugly, there are certainly
better ways to go about it.

Cool, right??
-------------

Okay, well maybe not *cool*, but learning about this definitely
helped the ReactiveUI unit tests themselves become much more
reliable and run way faster once I rewrote them to take advantage
of TestScheduler - those Sleeps really add up! If you want to learn
more about TestScheduler, Wes Dyer and Jeffrey Van Gogh from the Rx
team talk about it in-depth here:
[Wes Dyer and Jeffrey Van Gogh: Rx Virtual Time](http://channel9.msdn.com/Shows/Going+Deep/Wes-Dyer-and-Jeffrey-Van-Gogh-Inside-Rx-Virtual-Time)

Watching DependencyProperties using ReactiveUI
Watching DependencyProperties in WPF is easy...
-----------------------------------------------

One of the things that is pretty useful in XAML-based frameworks
like WPF and Silverlight when working in the codebehind is being
able to be notified when a DependencyProperty changes. In the
ViewModel, we have a different mechanism called
INotifyPropertyChanged to accomplish this, but DependencyProperties
are still an important part of WPF/Silverlight.

Let's see how we would do this in WPF - it's fairly
straightforward:

...but really ugly in Silverlight
---------------------------------

Unfortunately, this isn't possible in Silverlight - it's only
possible to wire up a *single* callback, and it can only be done by
the class that actually creates the DependencyProperty. This might
work fine in some scenarios, but something less tightly coupled is
often needed.

The solution is to use Attached Properties, as described
[Anoop Madhusudanan's blog post](http://amazedsaint.blogspot.com/2009/12/silverlight-listening-to-dependency.html)
- register an attached property, then hook *that* change
notification.

ReactiveUI now does this for you
--------------------------------

In ReactiveUI as of v2.0, there is a new method called
ObservableFromDP - this method works similarly to the ViewModel's
ObservableFromProperty, but with less syntactic noise:

Of course, since it's an Observable and not an event handler, all
of the power of Rx.NET applies to this as well. Nothing
revolutionary, but definitely makes things easier!

WCF.AsParallel() using ReactiveUI and Rx.NET
Select.AsParallel() for the Web
-------------------------------

I've mentioned it before, but SelectMany is the secret to using web
services. Check out
[this previous article on Web Services](http://blog.paulbetts.org/index.php/2010/09/26/calling-web-services-in-silverlight-using-reactivexaml/)
if you're new to using SelectMany with web services. Here's the
easy way to understand it in two lines, in terms of PowerShell /
\*nix pipes:

The cool thing about Rx.NET is that it makes it easy to write
completely non-blocking calls a-la node.js. If you've ever used the
*BeginXXXX/EndXXXX* calls, you know that it's not particularly easy
to use once you get into more complex examples, but that's one of
the advantages of Rx.NET - being able to take a bunch of
asynchronous operations and sews them together in a sane way. It's
easy to write, but what actually *happens* when we run it? Let's
see what code runs where:

![image](http://blog.paulbetts.org/wp-photos/SelectMany.png)

It's easy to be *way* too parallel
----------------------------------

Non-blocking web service calls take almost no time to execute -
this is one of its big advantages, but it also means we can go
through the input array really quickly. In effect, this means that
if we have an array with 100 elements,
**we will end up issuing 100 WCF requests at the same time!**

Not only is this not friendly to the web server on the other end,
it isn't possible - WCF will throttle your requests to 5 concurrent
requests by default, and fail the rest. We need a way to keep a
"pending queue", run a few at a time, and when each one completes,
pull a few more from the pending queue.

CachedSelectMany throttles concurrency
--------------------------------------

Let's see how the diagram looks like when we use `CachedSelectMany`
instead of SelectMany - from a code standpoint, CachedSelectMany
can simply be substituted in places where you use SelectMany with a
web service. CachedSelectMany internally uses a class called
ObservableAsyncMRUCache to manage concurrency. Despite the fact
that calls can be queued, your code doesn't actually wait - you
just won't be called back until the call completes.

![image](http://blog.paulbetts.org/wp-photos/CachedSelectMany.png)  
*Bar has to wait in line before it can run*

ReactiveUI Message Bus - decoupling objects using the
publish/subscribe pattern
Message buses allow us to decouple code
---------------------------------------

If you've used the MVVM pattern enough, you'll sometimes find that
you get "stuck" - you need to access a certain ViewModel (usually
the "main" ViewModel), but at the point you need it, it's too far
removed from the context that you're using it in. Sometimes if the
object is a Singleton it's appropriate to think of something like
the
[Managed Extensibility Framework](http://en.wikipedia.org/wiki/Managed_Extensibility_Framework)
to get a reference.

Other times, a Messaging framework is more appropriate - ReactiveUI
provides an "Rx"-take on the Messenger (Publish/Subscribe) pattern.
With the Messenger pattern, we don't have to make all our
ViewModels related to each other, which is important for writing
testable code, since I can easily replace related objects with mock
objects.

ReactiveUI's MessageBus
-----------------------

ReactiveUI's MessageBus is based on the Type of the message object.
If the Message type isn't unique enough, an extra Contract string
can be provided to make the source unique.

There are three main methods that we're interested in:

-   **RegisterMessageSource(IObservable source)** - Register an
    IObservable as a message source - anything that is published on the
    IObservable gets published to the bus.
-   **SendMessage(message)** - Publish a single item on the bus
    instead of having to use an Observable.
-   **IObservable Listen()** - Get an Observable for the specified
    source - you can either Subscribe to it directly, or use any of the
    Rx operators to filter what you want.

Special support for singleton ViewModels
----------------------------------------

For ViewModels that *only have a single instance*, there are a
number of helper methods that make using the MessageBus easier -
use these methods instead of the above ones:

-   **RegisterViewModel(ViewModel)** - Registers a ViewModel
    object; call this in your ViewModel constructor.
-   **IObservable ListenToViewModel** - Listen for change
    notifications on a ViewModel object
-   **IObservable ViewModelForType** - Return the ViewModel for the
    specified type

New Release: ReactiveUI 2.2.1
What does ReactiveUI do?
------------------------

[ReactiveUI](http://www.reactiveui.net) is a M-V-VM framework like
MVVM Light or Caliburn.Micro, that is deeply integrated with the
Reactive Extensions for .NET. This allows you to write code in your
ViewModel that is far more elegant and terse when expressing
complex stateful interactions, as well as much simpler handling of
async operations.

ReactiveUI on Hanselminutes
---------------------------

Check out
[the recent Hanselminutes episode about the Reactive Extensions](http://hanselminutes.com/default.aspx?showID=271)
as well if you've got more time. Scott and I chat about some of the
ideas in RxUI and how we can take the ideas in the Reactive
Extensions and use RxUI to apply them to Silverlight and WPF apps.

What's New in ReactiveUI 2.2.1 - Now with 100% less Windows Phone crashes
-------------------------------------------------------------------------

This release is just a maintenance release - if you don't currently
have any issues with RxUI, there is no reason to upgrade. However,
there are two major fixes that were worth creating a new release
for:

-   **.NET 4.0 Client Profile** - by including
    System.Reactive.Testing into ReactiveUI.dll, we broke everyone
    using the Client profile with WPF. This is now fixed and future
    versions of RxUI will be built against the Client profile.
-   **WP7 Crashes** - if you tried to use RxUI with WP7, you would
    receive a TypeLoadException whenever a type was instantiated, or
    possibly a XamlParseException telling you something to the effect
    of "MainWindow class does not exist". This issue is now fixed!

Breaking Change: Introducing ReactiveUI.Testing
-----------------------------------------------

To facilitate fixing the first bug above, a new Assembly / NuGet
package has been introduced, "ReactiveUI.Testing.dll /
ReactiveUI-Testing" - this was originally in ReactiveUI Core, and
the libraries here help you write better unit tests for your
applications (similar to Rx's System.Reactive.Testing). As a result
of this, you may need to add an extra package / library reference
to your project when you upgrade to 2.2.1.

Where can I find the library?
-----------------------------

On [NuGet](http://nuget.org)! The best way to install ReactiveUI
for a project is by installing the
[ReactiveUI package](http://nuget.org/Packages/Packages/Details/reactiveui-2-2-1-0)
for WPF/Silverlight projects, or
[ReactiveUI-WP7](http://nuget.org/Packages/Packages/Details/reactiveui-wp7-2-2-1-0)
for Windows Phone 7 projects.

If NuGet isn't your thing, you can also find the binaries on the
Github page:
[ReactiveUI 2.2.1.0.zip](https://github.com/downloads/xpaulbettsx/ReactiveUI/ReactiveUI%202.2.1.0.zip).

Using ReactiveUI with MVVM Light (or any other framework!)
ReactiveUI: Is it All or None?
------------------------------

No! RxUI provides all of the core components that you need to use
the M-V-VM pattern, but many people already have applications
written using some of the other great .NET MVVM frameworks, such as
MVVM Light or Caliburn.Micro - rewriting an entire codebase just to
use Rx is a total bummer.

Fortunately, in ReactiveUI 2.2, there have been several features
introduced in order to make using RxUI with existing frameworks
easier - you can try out RxUI on a per-page basis, instead of
rewriting your whole app.

There are really three core bits that are central to MVVM: a
ViewModel object, an ICommand implementation, and a
change-notifying Collection. Let's take a look at how we can Rx'ify
these three items.

Using MVVM Light alongside ReactiveUI
-------------------------------------

First, **File-\>New Project** and create a new "MVVM Light"
projects via the template. Then via NuGet (you are using NuGet,
right?), add a reference to the "ReactiveUI" project. Now, crack
open MainViewModel.cs. The **critical** thing to know is,
*all of RxUI's awesomeness are extensions onto an interface called IReactiveNotifyPropertyChanged*.
Ergo, we need to make MainViewModel implement this. But don't
panic, it's easy!

**Step 1:** Change the class definition to implement
IReactiveNotifyPropertyChanged.

`public class MainViewModel : ViewModelBase, IReactiveNotifyPropertyChanged`
**Step 2:** Add a field of type "MakeObjectReactiveHelper", and
initialize it in the constructor:

`MakeObjectReactiveHelper _reactiveHelper;  public MainViewModel() {     _reactiveHelper = new MakeObjectReactiveHelper(this);     /* More stuff */ }`
**Step 3:** Paste in the following code at the end of your class,
which just uses \_reactiveHelper to implement the entire
interface:

An important caveat about MakeObjectReactiveHelper
--------------------------------------------------

One thing that probably won't affect you, but it might:
MakeObjectReactiveHelper doesn't properly completely implement
IReactiveNotifyPropertyChanged unless you are also implementing
INotifyPropertyChang**ing** - most ViewModel implementations don't
(in fact, the interface doesn't even exist in Silverlight or WP7).
This means that in certain circumstances when you use the WhenAny
or ObservableForProperty with a deep path (i.e. x.Foo.Bar.Baz), you
may get duplicate notifications. In practice, this usually isn't a
big deal.

Watching ObservableCollections to create ViewModel collections
--------------------------------------------------------------

With RxUI 2.2, you can easily create a collection which tracks an
existing collection, even if the source is an ObservableCollection.
Here's the syntax:

Creating ReactiveCommands like RelayCommands
--------------------------------------------

Unfortunately, the story for ICommand isn't as easy, you have to
wrap commands one-at-a-time in order to subscribe to them. Here's
how to do it:

`public static ReactiveCommand WrapCommand(ICommand cmd) {     return ReactiveCommand.Create(cmd.CanExecute, cmd.Execute); }`

<!-- vim: ts=4 sw=4 et : -->
