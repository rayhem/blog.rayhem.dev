+++
title = 'Immutability is King'
date = '2025-11-07'
tags = ['guide', 'software', 'examples', 'immutability', 'c++']
series = ['style']
topics = ['software']
featured = true
weight = 2
draft = true
+++

# Introduction

# Example 1: Building a string

The following snippet comes from the Sim3d codebase. It builds a string of command line arguments from user-supplied pieces:
```matlab
Arguments = "";

gameModeClass = "";
if self.SimulationaMode == '5'
    gameModeClass = "?game=/MathWorksSimulation/Blueprints/Sim3dViewerGameMode.Sim3dViewerGameMode_C";
end

if (~strcmp(self.Map,""))
	Arguments = Arguments.append(strcat(self.Map, gameModeClass ," ")); % M1
end
Arguments = Arguments.append(strcat("-ExecCmds=","""",strjoin(self.ExecCmds, ";"),""""," ")); % M2
if (~strcmp(self.CommandLineArgs,""))
	Arguments = Arguments.append(strcat(self.CommandLineArgs," ")); % M3
end
if (~strcmp(self.RenderOffScreenFlag,""))
	Arguments = Arguments.append(strcat(self.RenderOffScreenFlag," ")); %M4
end
pakDir = fullfile(userpath,"sim3d_project",string(sprintf('R%s', version('-release'))),sim3d.utils.internal.ScenesMapping.getExtendedPakName,"AutoVrtlEnv","Content","Paks");
Arguments = Arguments.append(strcat("-pakdir=","""",pakDir,"""")); %M5
```

Logically, this declares a string (Arguments) and uses the append method to add the pieces through five(!) mutations (marked M1 through M5 in the comments). Consider the following:
- When this Arguments string gets used, does the order of the pieces matter (e.g. does RenderOffScreenFlag need to come after Map)? 
- If the ordering does need to change (perhaps because of a bug), will this be clear from the diff?
- Some of the argument pieces use an if statement and some of them don’t. Why? What’s the essential difference between those pieces?
Starting with the question “How can we construct Arguments correctly so as to avoid mutating it?”, this code turned into the following:

```matlab
gameModeClass = "";
if self.SimulationaMode == '5'
    gameModeClass = "?game=/MathWorksSimulation/Blueprints/Sim3dViewerGameMode.Sim3dViewerGameMode_C";
end

mapArg = "";
if ~strcmp(self.Map,"")
	mapArg = self.Map + gameModeClass;
end

execCmdArg = sprintf("-ExecCmds=""%s""", join(self.ExecCmds, ";"));

pakDir = fullfile(userpath(), "sim3d_project", "R" + version('-release'), sim3d.utils.internal.ScenesMapping.getExtendedPakName(), "AutoVrtlEnv", "Content", "Paks");
pakDirArg = sprintf("-pakdir=""%s""", pakDir);

Arguments = join([mapArg, execCmdArg, pakDirArg, join(self.CommandLineArgs)]);
```

As a result, 
- This refactor makes better use of MATLAB’s string manipulation tools.
- There are fewer lines of code. This is likely to make code simpler.
- There are fewer if statements and therefore fewer branches to think through and test. The branching logic for gameModeClass and mapArg follows the same structure and is easier to understand at a glance.
- mapArg, execCmdArg, pakDirArg, and otherArgs can all be built independently.
- For instance, it is easy to verify that execCmdArg does not depend on mapArg.
- Arguments is ready for use immediately after construction, and the ordering of the argument pieces is constrained to one line (as opposed to throughout the function).
- (The RenderOffScreenFlag piece moved into CommandLineArgs elsewhere in the code outside of this snippet.)

Conceptually, little has changed: we still just assemble a string of arguments. However, it is vastly easier to see the “flow” here and understand the pieces & how they interact.

# Example 2: Class invariants

Consider the following C++ class:
```C++
class ExpensiveCalculator {
    // This is some expensive operation that returns a large nubmer of doubles
    std::vector<double> ExpensiveOperation() const;

  public:

    void Tick(double dt) {
        std::vector<double> values = ExpensiveOperation();
        /* Do something with values, like plotting or curve fitting */
    }
};
```
Here, the class has a pretty standard decomposition of responsibilities: some function, `ExpensiveOperation`, generates data, and another function, `Tick`, calls it when it needs that data. We've made types explicit to indicate how things flow, and, because we know something about the implementation, we can even mark `ExpensiveOperation` as `const` for a little extra safety. Simple, right?

Not so fast! If you think back to your C++ fundamentals, every time `ExpensiveOperation` gets called it allocates and then returns a *new* vector. This dosen't cause much trouble if that vector only has a handful of items (though you still have to do a relatively expensive heap allocation), but if `ExpensiveOperation` produces a *hundred thousand* values at every pass through a tight loop, you'll likely start to see some performance issues. So, to avoid at least the cost to allocate each and every returned vector, let's add a payload:
```C++
class BetterExpensiveCalculator {
    // This field gets initialized once (in the class constructor)
    std::vector<double> payload;
    
    // ExpensiveOperation has access to the payload through this-> and won't
    // have to allocate each time it gets called, only if the payload grows or
    // shrinks. Performance will *really* suffer if you also return a vector
    // (requires copying payload), and it doesn't make much sense to return a
    // reference (as anything in the class can already access payload)
    void ExpensiveOperation();

  public:

    void Tick(double dt) {
        ExpensiveOperation();
        /* Do something with this->payload, like plotting or fitting */
    }
}
```
This could run faster[^1], but we've had to sacrifice _both_ our nice, strict typing _and_ `const`-correctness. For `ExpensiveOperation` to have any use at all, it _must_ mutate some other data within the instance as it now takes no arguments and returns no values. But this comes with a host of problems:
- We cannot anticipate what data changes from the signature alone—you’ll have to look at the (potentially complicated) implementation to figure it out
- We can’t guarantee we didn't accidentally change something else
- (Relatedly, the testing surface for this function has expanded to potentially "the enire object". A rigorous test would have to verify _everything_ remains in a consistent state after calling `ExpensiveOperation`.)

A major bummer.

```C++
class BestExpensiveCalculator {
    // The payload remains the same; a pile of data that persists for performance
    std::vector<double> payload;

public:
    // 
    void ExpensiveOperation(std::vector<double> &values) const;

    void Tick(double dt) {
        ExpensiveOperation(this->payload);
    }
};
```
Here, the data `ExpensiveOperation` is supposed to change has become explicit:
- The function cannot mutate be a class member (because the method has become `const`).
- It is safe to make this function public. Calling it from outside of the class cannot change the instance's values and therefore cannot change its behavior.
- The function still does not return a value (as it’s declared `void`). Consequently, the only thing that can change as a result of calling this function is whatever gets passed by non-const reference. When `Tick` invokes `ExpensiveOperation`, it passes `this->payload` like any other value. That `this->payload` belongs to the instance is incidental.

[^1]: Conventional wisdom applies: benchmark before prematurely optimizing.
