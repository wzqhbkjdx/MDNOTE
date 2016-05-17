###Scroller

View类的方法是个空函数，实现自定义View的滑动就需要手动去复写该方法

		/** 
 		* Called by a parent to request that a child update its values for mScrollX 
 		* and mScrollY if necessary. This will typically be done if the child is 
 		* animating a scroll using a {@link android.widget.Scroller Scroller} 
 		* object. 
 		*/
		public void computeScroll() 
		{ 
		}
		
那么：computeScroll()是怎样被调用的呢？看一下ViewGroup的源码

		@Override 
  		protected void dispatchDraw(Canvas canvas) { 
  
              ....... 
  
              ....... 
  
              ....... 
  
              ....... 
  
             for (int i = 0; i < childrenCount; i++) {
            while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
                final View transientChild = mTransientViews.get(transientIndex);
                if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                        transientChild.getAnimation() != null) {
                    more |= drawChild(canvas, transientChild, drawingTime);
                }
                transientIndex++;
                if (transientIndex >= transientCount) {
                    transientIndex = -1;
                }
            }
            int childIndex = customOrder ? getChildDrawingOrder(childrenCount, i) : i;
            final View child = (preorderedList == null)
                    ? children[childIndex] : preorderedList.get(childIndex);
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);
            }
        }
  
              ....... 
  
              ....... 
  
              ....... 
          }
ViewGroup 在dispatchDraw()函数中对它的每一个孩子调用drawChild()，再看看drawChild()方法

	protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }
    
调用的child的draw方法 在draw()方法中，会调用computeScroll();

	 private void smoothScrollBy(int dx, int dy) {
        mScroller.startScroll(getScrollX(), 0, dx, 0, 500);
        invalidate();
    }

    @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
            scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
            postInvalidate();
        }
    }

Scroller 的startScroll方法只是保存了一些参数，单靠startScroll是无法发生滑动的，关键在于invalidate方法，导致重绘，重绘的时候，View的draw方法会调用computeScroll，computeScroll是空方法，我们自己来实现，比如在computeScroll中调用scrollTo方法就可以使得View产生滑动效果，然后再调用postInvalidate方法来进行第二次重绘，如此反复，直到整个滑动过程结束 参见《Android开发艺术探索》第三章 至于mScroller.computeScrollOffset()的方法：

	public boolean computeScrollOffset() {
        if (mFinished) {
            return false;
        }

        int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    
        if (timePassed < mDuration) {
            switch (mMode) {
            case SCROLL_MODE:
                final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
                mCurrX = mStartX + Math.round(x * mDeltaX);
                mCurrY = mStartY + Math.round(x * mDeltaY);
                break;
            case FLING_MODE:
                final float t = (float) timePassed / mDuration;
                final int index = (int) (NB_SAMPLES * t);
                float distanceCoef = 1.f;
                float velocityCoef = 0.f;
                if (index < NB_SAMPLES) {
                    final float t_inf = (float) index / NB_SAMPLES;
                    final float t_sup = (float) (index + 1) / NB_SAMPLES;
                    final float d_inf = SPLINE_POSITION[index];
                    final float d_sup = SPLINE_POSITION[index + 1];
                    velocityCoef = (d_sup - d_inf) / (t_sup - t_inf);
                    distanceCoef = d_inf + (t - t_inf) * velocityCoef;
                }

                mCurrVelocity = velocityCoef * mDistance / mDuration * 1000.0f;
                
                mCurrX = mStartX + Math.round(distanceCoef * (mFinalX - mStartX));
                // Pin to mMinX <= mCurrX <= mMaxX
                mCurrX = Math.min(mCurrX, mMaxX);
                mCurrX = Math.max(mCurrX, mMinX);
                
                mCurrY = mStartY + Math.round(distanceCoef * (mFinalY - mStartY));
                // Pin to mMinY <= mCurrY <= mMaxY
                mCurrY = Math.min(mCurrY, mMaxY);
                mCurrY = Math.max(mCurrY, mMinY);

                if (mCurrX == mFinalX && mCurrY == mFinalY) {
                    mFinished = true;
                }

                break;
            }
        }
        else {
            mCurrX = mFinalX;
            mCurrY = mFinalY;
            mFinished = true;
        }
        return true;
    }
    
该方法的功能是随着时间的流逝，计算当前的scrollX和scrollY的值，相当于差之器的概念，根据时间流逝的百分比来计算，返回true，表示滑动未技术，返回false表示滑动结束

Scroller原理概括：Scroller本身并不能实现滑动，它需要配合View的computeScroll的方法才能完成弹性滑动的效果，它不断的让View重绘，而每一次重绘距滑动起始时间会有一个时间间隔，通过这个时间间隔，Scroller可以得到View当前的滑动位置，这样，View每次重绘都会导致View进行小幅度的移动，而多次的小幅度的移动就形成了弹性滑动。Scroller的设计很精妙，整个过程中，他对View没有丝毫的引用，甚至在它内部连计时器都没有。



