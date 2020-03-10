### react 渲染流程

#### ReactDOM.render 初次渲染

// TODO github md不支持uml图
```flow
s=>start: ReactDOM.render
o1=>operation: createFiberRoot
o2=>operation: updateContainer
o3=>operation: scheduleRootUpdate
o4=>operation: scheduleWork
o5=>operation: requestWork
o6=>operation: performSyncWork
o7=>operation: performWork
o8=>operation: performWorkOnRoot
o8=>operation: renderRoot
o9=>operation: workLoop
o10=>operation: performUnitOfWork
o11=>operation: updateHostRoot
o12=>operation: pushHostRootContext
o13=>operation: processUpdateQueue
o14=>operation: reconcileChildren
o15=>operation: mountChildFiber(ChildReconciler(false))
o16=>operation: ...
e=>end: end


s->o1->o2->o3->o4->o5->o6->o7->o8->o9->o10->o11->o12->o13->o14->o15->o16->e
```

#### 初次渲染后的update

```flow
s=>start: interactiveUpdates
o1=>operation: performSyncWork
o2=>operation: performWork
o3=>operation: performWorkOnRoot
in1=>condition: finishedWork是否为null
io=>operation: renderRoot
o4=>operation: completeRoot
o5=>operation: unstable_runWithPriority
o6=>operation: commitRoot
o7=>operation: commitAllHostEffect
o8=>operation: commitWork
o9=>operation: ...
in2=>condition: finishedWork是否为null
o10=>operation: nextUpdate

s->o1->o2->o3->in1(no)->o4->o5->o6->o7->o8
in1(yes)->io->o9->in2(no)->o4
in1(yes)->io->o9->in2(yes)->o10
```