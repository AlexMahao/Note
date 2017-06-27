## Transition 源码分析

通过`Transition`的学习，`Transtion`分为两种使用方式

-  通过`Scene`的切换监听，完成动画。
-  通过`Trasntion.beginDelayedTransition()`完成延时动画。

下面将对着两种方式进行分析。

### `Scene`实现过渡动画分析

`Scene`过渡动画分为两步:

```java
 Scene scene2 = Scene.getSceneForLayout(rootView, R.layout.scene2, this);
 TransitionManager.go(scene2, changeBounds);
```

-  看一下 `Scene.getSceneForLayout(ViewGroup sceneRoot, int layoutId, Context context)` 

```java 
 public static Scene getSceneForLayout(ViewGroup sceneRoot, int layoutId, Context context) {
 	// 利用tag存储Scene ， 避免重复创建
        SparseArray<Scene> scenes = (SparseArray<Scene>) sceneRoot.getTag(
                com.android.internal.R.id.scene_layoutid_cache);
        if (scenes == null) {
            scenes = new SparseArray<Scene>();
            sceneRoot.setTagInternal(com.android.internal.R.id.scene_layoutid_cache, scenes);
        }
        Scene scene = scenes.get(layoutId);
        if (scene != null) {
            return scene;
        } else {
            scene = new Scene(sceneRoot, layoutId, context);
            scenes.put(layoutId, scene);
            return scene;
        }
    }

```

-  `Transition.go()`：过渡动画实现的关键方法

```java
  public static void go(Scene scene, Transition transition) {
        changeScene(scene, transition);
    }
```
仅仅调用了` changeScene(scene, transition);`

```java
private static void changeScene(Scene scene, Transition transition) {
    		// 1，准备工作，计算初始值，并保存
            sceneChangeSetup(sceneRoot, transitionClone);
		    // 2，移除旧Scene,加载新Scene并添加到布局中
            scene.enter();
			// 3， 添加监听，启动动画
            sceneChangeRunTransition(sceneRoot, transitionClone);
        }
    }
```

####  1，准备工作，计算初始值，并保存

在`changeScene`分为三步，首先看第一步`sceneChangeSetup`,

```java
  private static void sceneChangeSetup(ViewGroup sceneRoot, Transition transition) {
        //  如果包含未完成的Scene动画，停止
        ArrayList<Transition> runningTransitions = getRunningTransitions().get(sceneRoot);
        if (runningTransitions != null && runningTransitions.size() > 0) {
            for (Transition runningTransition : runningTransitions) {
                runningTransition.pause(sceneRoot);
            }
        }
		// 计算旧Scene 的值，并保存
        if (transition != null) {
            transition.captureValues(sceneRoot, true);
        }

        // 通知之前的Scene退出
        Scene previousScene = Scene.getCurrentScene(sceneRoot);
        if (previousScene != null) {
            previousScene.exit();
        }
    }
```

关键点是` transition.captureValues(sceneRoot, true);`，该方法完成初始`Scene`的状态保存，以备过渡动画开始时使用3， 添加监听，启动动画

```java
void captureValues(ViewGroup sceneRoot, boolean start) {
    if ((mTargetIds.size() > 0 || mTargets.size() > 0)
            && (mTargetNames == null || mTargetNames.isEmpty())
            && (mTargetTypes == null || mTargetTypes.isEmpty())) {
            // 如果设置了target ，则只target保存View的属性状态
    } else {
        //否则直接递归调用保存所有的View的状态
        captureHierarchy(sceneRoot, start);
    }
  }
```

因为没有设置`target`，看一下`captureHierarchy()`方法：

```java
private void captureHierarchy(View view, boolean start) {
  	//...省略
  	 // 
     if (view.getParent() instanceof ViewGroup) {
        TransitionValues values = new TransitionValues();
        values.view = view;
        if (start) {
            // 计算初始`Scene`的状态
            captureStartValues(values);
        } else {
          	// 结束状态View 的保存
            captureEndValues(values);
        }
        // 添加过渡效果
        values.targetedTransitions.add(this);
        capturePropagationValues(values);
        if (start) {
          	// 保存初始状态值以及对应的View
            addViewValues(mStartValues, view, values);
        } else {
           	// 保存结束状态值以及对应的View
            addViewValues(mEndValues, view, values);
        }
    }
    if (view instanceof ViewGroup) {
       // 遍历所有的View保存状态
    }
}
```

在这个方法里，包含了开始状态和结束状态`View`的保存。通过`start == true`加以区分是开始和结束。在自定义`Transition`时，其中有三个方法需要重载实现的，其中开始状态和结束状态就是通过这个方法回调的。

因为在调用`transition.captureValues(sceneRoot, true);`是为true，所以此时只是保存View的初始状态。

到这里，第一步计算初始状态并保存结束，下面是第二步：移除旧Scene,加载新Scene并添加到布局中。

#### 2，移除旧Scene,加载新Scene并添加到布局中

看一下`scene.enter();`

```java
 public void enter() {
        if (mLayoutId > 0 || mLayout != null) {
            // 	移除ViewGroup下的所有View
            getSceneRoot().removeAllViews();
			// 加载新的布局
            if (mLayoutId > 0) {
                LayoutInflater.from(mContext).inflate(mLayoutId, mSceneRoot);
            } else {
                mSceneRoot.addView(mLayout);
            }
        }
		// 设置新的Scene到当前ViewGroup下
        setCurrentScene(mSceneRoot, this);
    }
```

首先移除rootView下的所有View，并加载新的`Scene`的布局到`rootView`中，并将新的`Scene`通过`tag`设置到`rootView`中。

#### 3， 添加监听，启动动画

看一下`sceneChangeRunTransition（）`方法

```java
 private static void sceneChangeRunTransition(final ViewGroup sceneRoot,
            final Transition transition) {
        if (transition != null && sceneRoot != null) {
            MultiListener listener = new MultiListener(transition, sceneRoot);
            //View从Window中dettach时的监听
            sceneRoot.addOnAttachStateChangeListener(listener);
          	// View重绘时的监听
            sceneRoot.getViewTreeObserver().addOnPreDrawListener(listener);
        }
    }
```

在这一步，构造了`MultiListener` ，通过设置监听可以看出，动画的相关操作放在了`MultiListener`中。

看一眼`onPreDraw`方法，该方法是重绘时的回调。

```java
  @Override
        public boolean onPreDraw() {
         	//...
          	// 计算结束的状态
            mTransition.captureValues(mSceneRoot, false);
            // 启动动画
            mTransition.playTransition(mSceneRoot);
            return true;
        }
    };
```

计算结束状态和计算开始状态的流程完成一样，主要看一下启动动画的方法。

```java
   void playTransition(ViewGroup sceneRoot) {
        // 匹配开始和结束View的对应关系.
        matchStartAndEnd(mStartValues, mEndValues);
		// 创建动画
        createAnimators(sceneRoot, mStartValues, mEndValues, mStartValuesList, mEndValuesList);
        // 启动动画
     	runAnimators();
    }
```



#### 总结

根据上面的源码分析，大致流程如下：

- 计算并保存开始时View的状态。
- 移除`rootView`下的`views`，并加载新的布局。
- 重绘时，计算并保存结束时View的状态。
- 最后启动过渡动画



###  `Trasntion.beginDelayedTransition()`延时动画源码分析

看一下该方法的实现：

```java
  public static void beginDelayedTransition(final ViewGroup sceneRoot, Transition transition) {
        	// 1， 初始准备，保存初始View的状态
            sceneChangeSetup(sceneRoot, transitionClone);
    		// 2，更新rootView下的Scene
            Scene.setCurrentScene(sceneRoot, null);
    		// 3，监听状态，启动动画
            sceneChangeRunTransition(sceneRoot, transitionClone);
        }
    }
```

通过这个流程，会发现和`Scene`的流程大致相同，唯一不同便是第二步。

`Scene`实现过渡动画第二步时，先移除了所有`View`，然后加载新布局，最后更新`rootView`的状态。

而对于延时动画，不需要加载新的布局，所以省略了移除和添加`View`的过程。



#### 总结

- 计算并保存开始时View的状态。
- 重绘时，计算并保存结束时View的状态。
- 最后启动过渡动画

















