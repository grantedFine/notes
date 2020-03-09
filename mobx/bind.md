### mobx 实现双向绑定

### 以computed为例
```flow
s=>start: computed.get
o1=>operation: computeValue
o2=>operation: 将computedValue赋值给global.trackingDerivation
o3=>operation: computedValue.derivation call, 也就是会call @computed修饰的func, 这个func会调用到其他observableValue的get function
o4=>operation: observableValue.get
o5=>operation: atom.get
o6=>operation: reportObservered,这一步会把引用到的observableValue加入到trackingDerivation的newObserving数组,实现了将computedValue引用到的其他被观察数据绑定到ComputedValue上.
o7=>operation: 下一次调用computed.get时会运行trackAndCompute
o7=>operation: trackDerivedFunction会重复之前的computedValue.derivation工作并将newObserving中的被观察者转移到computedValue.observing中,最后run bindDependencies
o8=>operation: bindDependencies, 这一步在每个不再引用的observing上移除了trackingDerivation即computedValue本身,又在新的每个observing上加上了trackingDerivation的observer,至此双向绑定完成.
o8=>operation: 当computed引用到的observableValue发生改变时,它会调用atom基类上的reportChanged方法
o9=>operation: propagateChanged被触发,observale上的所有observers被挨个标记需要更新.双向绑定生效.

s->o1->o2->o3->o4->o5->o6->o7->o8->o9
```

#### 相关的mobx中一些重要结构与方法, mobx所有类的基类 Atom, globalState, 还有ComputedValue, observableValue.

```ts
export class Atom implements IAtom {
    ...
    /**
     * Invoke this method to notify mobx that your atom has been used somehow.
     * Returns true if there is currently a reactive context.
     */
    public reportObserved(): boolean {
        return reportObserved(this)
    }

    /**
     * Invoke this method _after_ this method has changed to signal mobx that all its observers should invalidate.
     */
    public reportChanged() {
        startBatch()
        propagateChanged(this)
        endBatch()
    }
    ...
}

export class MobXGlobals {
    ...
    /**
     * Currently running derivation
     */
    trackingDerivation: IDerivation | null = null

    pendingUnobservations: IObservable[] = []

    /**
     * List of scheduled, not yet executed, reactions.
     */
    pendingReactions: Reaction[] = []
    ...
}


export class ComputedValue<T> implements IObservable, IComputedValue<T>, IDerivation {
    dependenciesState = IDerivationState.NOT_TRACKING
    observing: IObservable[] = [] // nodes we are looking at. Our value depends on these nodes
    newObserving = null // during tracking it's an array with new observed observers

    /**
     * Returns the current value of this computed value.
     * Will evaluate its computation first if needed.
     */
    public get(): T {
        if (this.isComputing) fail(`Cycle detected in computation ${this.name}: ${this.derivation}`)
        if (globalState.inBatch === 0 && this.observers.size === 0 && !this.keepAlive) {
            if (shouldCompute(this)) {
                this.warnAboutUntrackedRead()
                startBatch() // See perf test 'computed memoization'
                this.value = this.computeValue(false)
                endBatch()
            }
        } else {
            reportObserved(this)
            if (shouldCompute(this)) if (this.trackAndCompute()) propagateChangeConfirmed(this)
        }
        const result = this.value!

        if (isCaughtException(result)) throw result.cause
        return result
    }

    public set(value: T) {
        if (this.setter) {
            invariant(
                !this.isRunningSetter,
                `The setter of computed value '${
                this.name
                }' is trying to update itself. Did you intend to update an _observable_ value, instead of the computed property?`
            )
            this.isRunningSetter = true
            try {
                this.setter.call(this.scope, value)
            } finally {
                this.isRunningSetter = false
            }
        } else
            invariant(
                false,
                process.env.NODE_ENV !== "production" &&
                `[ComputedValue '${
                this.name
                }'] It is not possible to assign a new value to a computed value.`
            )
    }

    private trackAndCompute(): boolean {
        if (isSpyEnabled() && process.env.NODE_ENV !== "production") {
            spyReport({
                object: this.scope,
                type: "compute",
                name: this.name
            })
        }
        const oldValue = this.value
        const wasSuspended =
            /* see #1208 */ this.dependenciesState === IDerivationState.NOT_TRACKING
        const newValue = this.computeValue(true)

        const changed =
            wasSuspended ||
            isCaughtException(oldValue) ||
            isCaughtException(newValue) ||
            !this.equals(oldValue, newValue)

        if (changed) {
            this.value = newValue
        }

        return changed
    }

    computeValue(track: boolean) {
        this.isComputing = true
        globalState.computationDepth++
        let res: T | CaughtException
        if (track) {
            res = trackDerivedFunction(this, this.derivation, this.scope)
        } else {
            if (globalState.disableErrorBoundaries === true) {
                res = this.derivation.call(this.scope)
            } else {
                try {
                    res = this.derivation.call(this.scope)
                } catch (e) {
                    res = new CaughtException(e)
                }
            }
        }
        globalState.computationDepth--
        this.isComputing = false
        return res
    }

}

export function reportObserved(observable: IObservable): boolean {
    const derivation = globalState.trackingDerivation
    if (derivation !== null) {
        /**
         * Simple optimization, give each derivation run an unique id (runId)
         * Check if last time this observable was accessed the same runId is used
         * if this is the case, the relation is already known
         */
        if (derivation.runId !== observable.lastAccessedBy) {
            observable.lastAccessedBy = derivation.runId
            // Tried storing newObserving, or observing, or both as Set, but performance didn't come close...
            derivation.newObserving![derivation.unboundDepsCount++] = observable
            if (!observable.isBeingObserved) {
                observable.isBeingObserved = true
                observable.onBecomeObserved()
            }
        }
        return true
    } else if (observable.observers.size === 0 && globalState.inBatch > 0) {
        queueForUnobservation(observable)
    }
    return false
}


export function trackDerivedFunction<T>(derivation: IDerivation, f: () => T, context: any) {
    // pre allocate array allocation + room for variation in deps
    // array will be trimmed by bindDependencies
    changeDependenciesStateTo0(derivation)
    derivation.newObserving = new Array(derivation.observing.length + 100)
    derivation.unboundDepsCount = 0
    derivation.runId = ++globalState.runId
    const prevTracking = globalState.trackingDerivation
    globalState.trackingDerivation = derivation
    let result
    if (globalState.disableErrorBoundaries === true) {
        result = f.call(context)
    } else {
        try {
            result = f.call(context)
        } catch (e) {
            result = new CaughtException(e)
        }
    }
    globalState.trackingDerivation = prevTracking
    bindDependencies(derivation)
    return result
}

function bindDependencies(derivation: IDerivation) {
    // invariant(derivation.dependenciesState !== IDerivationState.NOT_TRACKING, "INTERNAL ERROR bindDependencies expects derivation.dependenciesState !== -1");
    const prevObserving = derivation.observing
    const observing = (derivation.observing = derivation.newObserving!)
    let lowestNewObservingDerivationState = IDerivationState.UP_TO_DATE

    // Go through all new observables and check diffValue: (this list can contain duplicates):
    //   0: first occurrence, change to 1 and keep it
    //   1: extra occurrence, drop it
    let i0 = 0,
        l = derivation.unboundDepsCount
    for (let i = 0; i < l; i++) {
        const dep = observing[i]
        if (dep.diffValue === 0) {
            dep.diffValue = 1
            if (i0 !== i) observing[i0] = dep
            i0++
        }

        // Upcast is 'safe' here, because if dep is IObservable, `dependenciesState` will be undefined,
        // not hitting the condition
        if (((dep as any) as IDerivation).dependenciesState > lowestNewObservingDerivationState) {
            lowestNewObservingDerivationState = ((dep as any) as IDerivation).dependenciesState
        }
    }
    observing.length = i0

    derivation.newObserving = null // newObserving shouldn't be needed outside tracking (statement moved down to work around FF bug, see #614)

    // Go through all old observables and check diffValue: (it is unique after last bindDependencies)
    //   0: it's not in new observables, unobserve it
    //   1: it keeps being observed, don't want to notify it. change to 0
    l = prevObserving.length
    while (l--) {
        const dep = prevObserving[l]
        if (dep.diffValue === 0) {
            removeObserver(dep, derivation)
        }
        dep.diffValue = 0
    }

    // Go through all new observables and check diffValue: (now it should be unique)
    //   0: it was set to 0 in last loop. don't need to do anything.
    //   1: it wasn't observed, let's observe it. set back to 0
    while (i0--) {
        const dep = observing[i0]
        if (dep.diffValue === 1) {
            dep.diffValue = 0
            addObserver(dep, derivation)
        }
    }

    // Some new observed derivations may become stale during this derivation computation
    // so they have had no chance to propagate staleness (#916)
    if (lowestNewObservingDerivationState !== IDerivationState.UP_TO_DATE) {
        derivation.dependenciesState = lowestNewObservingDerivationState
        derivation.onBecomeStale()
    }
}

export function propagateChanged(observable: IObservable) {
    // invariantLOS(observable, "changed start");
    if (observable.lowestObserverState === IDerivationState.STALE) return
    observable.lowestObserverState = IDerivationState.STALE

    // Ideally we use for..of here, but the downcompiled version is really slow...
    observable.observers.forEach(d => {
        if (d.dependenciesState === IDerivationState.UP_TO_DATE) {
            if (d.isTracing !== TraceMode.NONE) {
                logTraceInfo(d, observable)
            }
            d.onBecomeStale()
        }
        d.dependenciesState = IDerivationState.STALE
    })
    // invariantLOS(observable, "changed end");
}

```