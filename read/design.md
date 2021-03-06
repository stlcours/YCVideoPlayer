#### 面向对象优化介绍
- 01.面向对象6大原则介绍
- 02.单一指责的演变和迭代
- 07.视频状态View的演变







### 07.视频状态View的演变
- 关于视频状态view都有那些呢？先简单列举一下常见的视图……
    - 视频加载视图，视频播放错误视图，视频重试视图，视频播放完成视图，视频网络变化视图，视频投屏视图，视频倍速播放视图，视频滑动缩略图，视频弹幕视图，视频清晰度视图，视频手势指导视图，视频手势滑动改变音量和亮度视图，视频广告视图，视频下载视图……这个仅仅是一部分，有没有发现真的特别多，而且这么多视频的显示和隐藏状态，随着后期的迭代，如何降低维护的成本也是值得思考的问题。
- 最刚开始的做法是，将视频加载loading，视频播放错误，视频重试，网络变化等布局写到了一起，代码如下所示。
    - 这个是最原始的做法，让每种状态的布局都写出来，然后放到一起。根据播放的逻辑进行动态VISIBLE和GONE处理。
    ```
    //省略部分代码
    <?xml version="1.0" encoding="utf-8"?>
    <RelativeLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        
        <!--加载动画view-->
        <include layout="@layout/custom_video_player_loading"/>
        <!--改变播放位置-->
        <include layout="@layout/custom_video_player_change_position"/>
        <!--改变亮度-->
        <include layout="@layout/custom_video_player_change_brightness"/>
        <!--改变声音-->
        <include layout="@layout/custom_video_player_change_volume"/>
        <!--播放完成，你也可以自定义-->
        <include layout="@layout/custom_video_player_completed"/>
        <!--播放错误-->
        <include layout="@layout/custom_video_player_error"/>
        <!--播放重试-->
        <include layout="@layout/custom_video_player_reset"/>
    
    </RelativeLayout>
    ```
    - 然后是如何控制播放状态的呢？代码如下所示，后来发现如果状态页面过多的话，关于状态视图的展示和隐藏有点麻烦。当然可以实现，但是容易出错……
    ```
    /**
     * 当播放状态发生改变时
     * @param playState 播放状态：
     */
    @SuppressLint("SetTextI18n")
    @Override
    public void onPlayStateChanged(@ConstantKeys.CurrentState int playState) {
        switch (playState) {
            case ConstantKeys.CurrentState.STATE_IDLE:
                break;
            //播放准备中
            case ConstantKeys.CurrentState.STATE_PREPARING:
                startPreparing();
                break;
            //播放准备就绪
            case ConstantKeys.CurrentState.STATE_PREPARED:
                startUpdateProgressTimer();
                //取消缓冲时更新网络加载速度
                cancelUpdateNetSpeedTimer();
                break;
            //正在播放
            case ConstantKeys.CurrentState.STATE_PLAYING:
                statePlaying();
                break;
            //暂停播放
            case ConstantKeys.CurrentState.STATE_PAUSED:
                statePaused();
                break;
            //正在缓冲(播放器正在播放时，缓冲区数据不足，进行缓冲，缓冲区数据足够后恢复播放)
            case ConstantKeys.CurrentState.STATE_BUFFERING_PLAYING:
                stateBufferingPlaying();
                break;
            //暂停缓冲
            case ConstantKeys.CurrentState.STATE_BUFFERING_PAUSED:
                stateBufferingPaused();
                break;
            //播放错误
            case ConstantKeys.CurrentState.STATE_ERROR:
                stateError();
                break;
            //播放完成
            case ConstantKeys.CurrentState.STATE_COMPLETED:
                stateCompleted();
                break;
            default:
                break;
        }
    }
    ```
- 迭代留下的遗留问题
    - 如果是添加新的状态页面，则需要在布局中又要添加一个类型，后期的布局变得不好维护。
    - 关于新的状态页面的展示和隐藏，又需要看视频的播放逻辑，然后动态显示和隐藏。如果状态页面很多，则变得不好维护。
    - 比如针对视频播放异常，视频播放重试，需要定制不同的UI。那么又该如何操作，是不是缺乏一定的灵活性。
- 根据面向对象思想的优化
    - 针对视频加载loading，视频播放错误，视频重试，网络变化，视频试看等n中状态页面是否能够用状态管理器管理。这个管理器功能单一，主要是处理切换视频状态view的显示和隐藏逻辑。
    - n中状态页面必须要单独抽出来，不要统一写在布局中，避免后期代码维护臃肿。比如播放错误view，能否把它封装成一个ErrorView对象，然后只需要加载自己布局即可，至于设置错误的文案，可以用set方法暴露出来。
    - 针对n个状态页面，可能会有触发事件。比如视频重试，点击会重新加载视频；比如网络从wifi切换4g，点击会进行4g播放视频。点击事件需要暴露给开发者处理。
- 然后看看优化后的伪代码思路
    - 下面仅仅是针对加载错误的视图View，可以看出该类功能单一，也很灵活，针对文案相关内容可以直接设置。点击事件也暴露给外部开发者。然后以此类推，n个状态视图都可以定义为这样的类。
    ```
    /**
     * 错误提示对话框。出错的时候会显示。
     */
    public class ErrorView extends RelativeLayout {
    
        private static final String TAG = ErrorView.class.getSimpleName();
        //错误信息
        private TextView mMsgView;
        //错误码
        private TextView mCodeView;
        //重试的图片
        private View mRetryView;
        //重试的按钮
        private TextView mRetryBtn;
    
        private OnRetryClickListener mOnRetryClickListener = null;//重试点击事件
    
        public ErrorView(Context context) {
            super(context);
            init();
        }
    
    
        public ErrorView(Context context, AttributeSet attrs) {
            super(context, attrs);
            init();
        }
    
    
        public ErrorView(Context context, AttributeSet attrs, int defStyleAttr) {
            super(context, attrs, defStyleAttr);
            init();
        }
    
        private void init() {
            LayoutInflater inflater = (LayoutInflater) getContext()
                                      .getApplicationContext().getSystemService(Context.LAYOUT_INFLATER_SERVICE);
            Resources resources = getContext().getResources();
    
            View view = inflater.inflate(R.layout.alivc_dialog_error, null);
            addView(view);
    
            mRetryBtn = (TextView) view.findViewById(R.id.retry_btn);
            mMsgView = (TextView) view.findViewById(R.id.msg);
            mCodeView = (TextView) view.findViewById(R.id.code);
            mRetryView = view.findViewById(R.id.retry);
            //重试的点击监听
            mRetryView.setOnClickListener(new OnClickListener() {
                @Override
                public void onClick(View v) {
                    if (mOnRetryClickListener != null) {
                        mOnRetryClickListener.onRetryClick();
                    }
                }
            });
        }
    
        /**
         * 更新提示文字
         * @param errorCode 错误码
         * @param errorEvent 错误事件
         * @param errMsg 错误码
         */
        public void updateTips(int errorCode, String errorEvent, String errMsg) {
            mMsgView.setText(errMsg);
            mCodeView.setText(getContext().getString(R.string.alivc_error_code) + errorCode + " - " + errorEvent);
        }
    
        /**
         * 更新提示文字,不包含错误码
         */
        public void updateTipsWithoutCode(String errMsg) {
            mMsgView.setText(errMsg);
            mCodeView.setVisibility(View.GONE);
        }
    
        /**
         * 重试的点击事件
         */
        public interface OnRetryClickListener {
            /**
             * 重试按钮点击
             */
            void onRetryClick();
        }
    
        /**
         * 设置重试点击事件
         * @param l 重试的点击事件
         */
        public void setOnRetryClickListener(OnRetryClickListener l) {
            mOnRetryClickListener = l;
        }
    }
    ```
    - 写到这里，n个不同视图状态的类定义好了，那么它们是怎么进行显示和隐藏呢？是如何添加到视频播放器上的呢。这个时候就需要用到一个状态管理的类，这里只是写了为代码。有没有发现这样操作的话，就特别灵活，而且在加载布局这块，主要当需要用到的时候，才会把视图addView到主视图中
    ```
    public class TipsView extends RelativeLayout{
    
        /**
         * 显示重播view
         */
        public void showReplayTipView() {
            if (mReplayView == null) {
                mReplayView = new ReplayView(getContext());
                mReplayView.setOnReplayClickListener(onReplayClickListener);
                addSubView(mReplayView);
            }
    
            if (mReplayView.getVisibility() != VISIBLE) {
                mReplayView.setVisibility(VISIBLE);
            }
        }
    
        /**
         * 隐藏重播的tip
         */
        public void hideReplayTipView() {
            if (mReplayView != null && mReplayView.getVisibility() == VISIBLE) {
                mReplayView.setVisibility(INVISIBLE);
            }
        }
    
        /**
         * 提示view中的点击操作
         */
        public interface OnTipClickListener {
            /**
             * 继续播放
             */
            void onContinuePlay();
    
            /**
             * 停止播放
             */
            void onStopPlay();
    
            /**
             * 重试播放
             */
            void onRetryPlay();
    
            /**
             * 重播
             */
            void onReplay();
    
            /**
             * 刷新sts
             */
            void onRefreshSts();
        }
    
        /**
         * 设置提示view中的点击操作 监听
         *
         * @param l 监听事件
         */
        public void setOnTipClickListener(OnTipClickListener l) {
            mOnTipClickListener = l;
        }
    }
    ```



