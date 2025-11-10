# Falloff in Detail

Audio falloff behavior is easily one of the more misunderstood concepts in
audio. Most people understand audio to follow the inverse-square law relative to
distance. And while this is technically correct, it is also very misleading.

## Energy Dispersion

First, let’s catch up on what the inverse-square law even means.

Imagine you have a sound pulse of energy emanating from a small sphere or radius
*r*. The energy is spread out over the surface of the sphere. The formula for
the surface of a sphere is 4*πr*². Because this formula uses the squared radius,
at double the radius, the surface area is 4x larger. That means that as the
sound travels outwards in all direction, once it reaches 2*r* from the emission
sphere’s center, the energy now needs to be spread out over a 4x larger area.
This means that the amount of energy focused at any given point in space shrinks
by 4x every time the sound travels double the distance. Thus, the sound
intensity shrinks by the square distance. This is the inverse-square law.

If the sound pulse is continuous over time, then it represents a power source.
And therefore, the sound power flowing at any specific point in 3D space also
follows the inverse-square law.

## Pressure Dispersion

The problem here is that audio devices don’t actually work with sound intensity.
Instead, they work with sound pressure. That’s because sound pressure can be
converted into an electrical form and back. In this way, sound pressure is
directly related to voltage. If you know electrical circuitry, then you know
that P = V²/R. From this, it is not hard to see that voltage, and therefore
sound pressure, is proportional to the square-root of power. Thus, the
square-root of the inverse-square law is the inverse distance law. That means
that sound pressure decreases by distance, rather than distance squared.

That’s what most people mess up, including in Myri until 0.14.2.

If you aren’t convinced by the electrical conversion, and want to reason about
this mechanically, you may struggle to find the derivation of this law in an
intuitive sense. Allow me to provide one that is in no way mathematically
rigorous.

In a sound wave, little air particles are vibrating back and forth. Over a
longer period of time, their average position does not change. Therefore, each
particle moves backwards as much as it moves forward. A behavior like this can
always be characterized as a bunch of sine waves stacked on top of each other.
For simplicity, let’s assume we only have a single sine wave.

Our particle is moving back and forth in a sine wave pattern. A sine wave has an
amplitude. This is the distance amplitude. However, the calculus derivative of a
sine wave is also a sine wave of the same amplitude. So the amplitude of the
position the particle moves is also the amplitude of the velocity the particle
is moving in. And as you might guess, the amplitude of the acceleration is also
the same.

Next, we have the formula for kinetic energy:

E = *mv*²/2

Sound is all kinetic energy, and it doesn’t transfer energy into its
surroundings easily. We can generally assume conservation of energy here.
However, at 2x distance, the sound wavefront now covers 4x the area. At uniform
density, that means 4x particles and 4x mass. To not add new energy, the
velocity needs to be half the amount (reminder, velocity is squared). From this,
we can see that velocity follows an inverse law, rather than the inverse square
law. The amplitude any individual particle is vibrating at twice the distance
away from the sound source will be half.

Lastly, pressure is force per area. Throw in F = ma, we can see force is
proportional to the acceleration. Therefore, the amplitude of the force is
proportional to the amplitude of the acceleration, which is the same as the
amplitude of the velocity. Thus, pressure follows the same inverse-law as
particle velocity.

## Falloff in Practice

Sound pressure decreases by half every time you double distance. But that also
means sound pressure doubles whenever the distance is halved. If the listener
gets very close to a sound source, sound pressure could rocket up to infinity.
That’s not good.

To get around this, Myri defines the original sound pressure of an audio source
as the sound pressure at the *Inner Range*. Anything closer than the *Inner
Range* is treated as having the same sound pressure as at the *Inner Range*.

This has another side-effect. Because the *Inner Range* represents the original
spherical radius of the source, increasing it also increases the distance that
is considered “doubled” where then the sound pressure is at half the original
volume. This means that increasing the *Inner Range* slows down the falloff over
distance.

The *Outer Range* and *Range Fade Margin* are simply a taper effect to reduce
the remaining sound pressure at that distance down to zero. They do not have any
effect on the falloff up until `outerRange – rangeFadeMargin` distance away from
the audio source. Beyond that point, the sound pressure decreases by more than
the inverse-law.
