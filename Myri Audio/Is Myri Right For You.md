# Is Myri Right for You?

This is a question that gets asked every once in a while. Many devs struggle
with the differences between various audio solutions. This guide will break
things down.

## Is Your Game a Heavy Simulation Game?

Some types of games involve simulating lots of entities, perhaps in the tens or
hundreds of thousands. If each of those entities also needs to emit sound, then
Myri is a strong contender!

Myri’s automatic out-of-the-box voice combining makes management of high numbers
of simple audio emitters painless. I’m not aware of any other established audio
solution which has this technology.

## Are Your Audio Needs Simple?

Myri is a very simple audio solution with only a few essentials.

-   Play looping and non-looping audio sources
-   Compress audio clips to a memory ratio of 8:1
-   Perform basic channel mixing with built-in limiters
-   Perform some amount of listener-relative spatialization (fancier than
    panning, not as fancy as HRTFs)
-   Apply uniform or directed audio
-   Apply a basic constant tape-warp by scaling sampling rate and interpolating
    (linked time scaling and pitch shifting)

If this is enough for your project, and your project makes heavy use of Unity’s
ECS, then Myri may be a good fit. Audio sources as pure ECS entities means you
don’t need to build your own bridge to Unity’s Game Object-based audio solution.

## Do You Need More Than What Myri Provides?

If your project doesn’t match the above, then Myri may not be a good fit. But
then that begs the question, what do you use instead?

Industry-standard middleware such as FMOD Studio and Wwise can provide much more
than what Unity provides out-of-the-box with Game Objects. These solutions can
scale to hundreds of audio emitters, each with complex effect chains and a suite
of professional tools to perfect your soundscape. And they offer thread-safe
Burst-compatible APIs, making them friendly for ECS projects. The main downside
of them is that their licensing costs can be quite a challenge to swallow if
your project doesn’t have guaranteed launch visibility.

If licensing is a problem, then you will be looking at either using Game Object
audio, or perhaps you might consider mixing Game Object audio with Myri. Unity’s
built-in Game Object audio provides proper pitch shifting, custom distance
falloff curves, reverb, effect chains, and custom mixing profiles. However, it
is quite limited in how many audio sources it can handle at once.

Aside from Unity’s built-in audio solution, several other free audio middleware
solutions exist that have GameObject integration. Steam Audio is a more popular
choice, but there are other options too. Unfortunately, many of these solutions
don’t provide a Burst-compatible API, instead focusing on the GameObject
workflow.

## Will Myri Offer More Features in the Future?

Ideally, Myri’s development will bring to life more powerful features,
eventually making it a one-stop shop for ECS audio. But the road ahead is long,
and it will require the help of community contributors to reach that goal.

The current task at hand is to migrate Myri’s backend from DSP Graph to a
Scriptable Audio Root Output built on Aux ECS. Once this migration is complete,
Myri will have a public API for executing code on the DSP thread, making it much
easier for people to develop their own effects and features.
