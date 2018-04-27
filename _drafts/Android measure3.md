安卓measure的原理和流程
==================
上回我们分析了触发ViewRootImpl启动重新measure的条件，以及ViewRootImpl发起measure和layout的函数doTraversal。这次，我们就研究一下doTraversal是怎么启动measure的。
  
    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }

好吧，我们被骗了，doTraversal没干啥特别的，它又调用了performTraversals()。我现在尴尬癌都要犯了，为了避免我找个地缝钻下去，我们还是赶紧看一下performTraversals()都干了些什么吧。
  
