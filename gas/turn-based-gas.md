# Turn-based Gameplay Ability System
GAS provides a great framework for RPG systems of many styles. However it has no inherent support for turn-based games. This document is intended to describe some approaches I have experimented with to use GAS for a turn-based game.
My goal with experimenting with these approaches is to find a method that has feature parity with real-time GameplayEffect usage, while being minimally disruptive.

This page is a work in progress and currently only servers as a scaffold to remind me of approaches I have tried, their limitations, and possible areas of exploration.

## Foreword
- I haven't used any solutions here in a shipped product, only for my own hobby projects and learning. No guarantees are provided on their scalability etc.
- I have been working on this project on-and-off over several years and have scrapped a lot of code, so some implementations are going to be a bit fuzzy on detail as I experiment and fill them out.
- Suggestions, notes and outside experience are more than welcome

## Table of Contents
1. Custom AbilityTimerManager

## Custom AbilityTimerManager
Forum user BinaryBard describes an approach [here](https://forums.unrealengine.com/t/gameplay-ability-system-turn-based/126620/6) in which they override the default TimerManager to require manual execution in GameplayEffects.
This is further expanded [here](https://juejin.cn/post/7359391403162353683) in a Chinese blog post (I don't want to rewrite somebody else's work and claim credit, but the blog images contain English code and Google Translate does a pretty good job with the text content).

This approach requires forking the GameplayAbilitySystem plugin. I haven't personally tried it as I have been hoping to find an approach that can build on top of the existing systems.

## Custom GameplayEffectContext
The `FGameplayEffectContext` struct can be overridden without modifications to the underlying GAS plugin. In this FCustomGameplayEffectContext you can then add properties to track e.g. number of turns we want to apply, number of applications remaining, do we apply at the start or end of the turn etc.

A turn-based GameplayEffect can then expose these same properties and initialise them on the EffectContext when applied. This GameplayEffect should have an `Infinite` duration and a period of 0 to prevent it activating on interval (I believe the period of 0 had to be set programmatically, as the editor validation prevents a period of 0 otherwise).

Your subclassed AbilitySystemComponent can then register events for turn start/end and manually execute the Effect using `void ExecutePeriodicEffect(FActiveGameplayEffectHandle	Handle);`, and clean up the GE once it has exhausted executions.

### Cons
The main issue I found with this approach before I pivoted to others was that you lost GAS's implicit handling of stacking GE's and associated refresh policies (as your duration is now tracked by properties on the GameplayEffectContext). It might still be able to make this work, but I suspect a lot of edge cases would get in the way.
