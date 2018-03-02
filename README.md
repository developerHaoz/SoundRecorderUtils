### 前言
> 最近项目中需要用到录音的功能，搞定需求之后，花了些时间封装成一个录音的工具包，分享给大家，需要源码的 [点击这里](https://github.com/developerHaoz/SoundRecorderUtils)。

先贴个效果图给大家看一下，看看这个录音包的功能。

![SoundRecorderUtils.gif](http://upload-images.jianshu.io/upload_images/4334738-d1288c24e6707d59.gif?imageMogr2/auto-orient/strip)


### 一、实现录音的 Service
-----
这个类可以说是这个包的核心了，如果理解了这个 `Service`，录音这一块基本就没什么问题了。

录音主要是利用 `MediaRecoder` 这个类，进行声音的记录，接下来我们一起来看看具体的实现。
```
public class RecordingService extends Service {

    private String mFileName;
    private String mFilePath;

    private MediaRecorder mRecorder;

    private long mStartingTimeMillis;
    private long mElapsedMillis;

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        startRecording();
        return START_STICKY;
    }

    @Override
    public void onDestroy() {
        if (mRecorder != null) {
            stopRecording();
        }
        super.onDestroy();
    }

    // 开始录音
    public void startRecording() {
        setFileNameAndPath();

        mRecorder = new MediaRecorder();
        mRecorder.setAudioSource(MediaRecorder.AudioSource.MIC);
        mRecorder.setOutputFormat(MediaRecorder.OutputFormat.MPEG_4); //录音文件保存的格式，这里保存为 mp4
        mRecorder.setOutputFile(mFilePath); // 设置录音文件的保存路径
        mRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.AAC);
        mRecorder.setAudioChannels(1);
        // 设置录音文件的清晰度
        mRecorder.setAudioSamplingRate(44100);
        mRecorder.setAudioEncodingBitRate(192000);

        try {
            mRecorder.prepare();
            mRecorder.start();
            mStartingTimeMillis = System.currentTimeMillis();
        } catch (IOException e) {
            Log.e(LOG_TAG, "prepare() failed");
        }
    }
    
    // 设置录音文件的名字和保存路径
    public void setFileNameAndPath() {
        File f;

        do {
            count++;
            mFileName = getString(R.string.default_file_name)
                    + "_" + (System.currentTimeMillis()) + ".mp4";
            mFilePath = Environment.getExternalStorageDirectory().getAbsolutePath();
            mFilePath += "/SoundRecorder/" + mFileName;
            f = new File(mFilePath);
        } while (f.exists() && !f.isDirectory());
    }
   
    // 停止录音
    public void stopRecording() {
        mRecorder.stop();
        mElapsedMillis = (System.currentTimeMillis() - mStartingTimeMillis);
        mRecorder.release();

        getSharedPreferences("sp_name_audio", MODE_PRIVATE)
                .edit()
                .putString("audio_path", mFilePath)
                .putLong("elpased", mElapsedMillis)
                .apply();
        if (mIncrementTimerTask != null) {
            mIncrementTimerTask.cancel();
            mIncrementTimerTask = null;
        }

        mRecorder = null;
    }

}
```
可以看到在 `onStartCommand()` 里面有一个 `startRecording()` 方法，在外部启动这个 `RecordingService` 的时候，便会调用这个 `startRecording()` 方法开始录音。

在 `startRecording()` 方法中先调用了 `setFileNameAndPath` 方法，初始化了录音文件的名字和保存的路径，为了让每个录音文件都有唯一的名字，我调用 `System.currentMillis()` 拼接到录音文件的名字里面。
```
    public void setFileNameAndPath() {
        File f;

        do {
            count++;
            mFileName = getString(R.string.default_file_name)
                    + "_" + (System.currentTimeMillis()) + ".mp4";
            mFilePath = Environment.getExternalStorageDirectory().getAbsolutePath();
            mFilePath += "/SoundRecorder/" + mFileName;
            f = new File(mFilePath);
        } while (f.exists() && !f.isDirectory());
    }
```

设置好了文件的名字和保存路径之后，对 `mRecorder` 进行一系列参数的设置，这个  `mRecorder` 是 `MediaRecorder` 的一个实例，专门用于录音的存储。
```
        mRecorder = new MediaRecorder();
        mRecorder.setAudioSource(MediaRecorder.AudioSource.MIC);
        mRecorder.setOutputFormat(MediaRecorder.OutputFormat.MPEG_4); //录音文件保存的格式，这里保存为 mp4
        mRecorder.setOutputFile(mFilePath); // 设置录音文件的保存路径
        mRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.AAC);
        mRecorder.setAudioChannels(1);
        // 设置录音文件的清晰度
        mRecorder.setAudioSamplingRate(44100);
        mRecorder.setAudioEncodingBitRate(192000);

        try {
            mRecorder.prepare();
            mRecorder.start();
            mStartingTimeMillis = System.currentTimeMillis();
        } catch (IOException e) {
            Log.e(LOG_TAG, "prepare() failed");
        }
```
设置好参数之后，启动 `mRecorder` 开始录音，可以看到启动 `mRecorder` 开始录音后，我还将当前的时间赋值给 `mStartingTimeMills`，这里主要是为了记录录音的时长，等到录音结束后再获取一次当前的时间，然后将两个时间进行相减，就能得到录音的具体时长了。

等到录音结束，停止服务后，便会回调 `RecordingService` 的 `onDestroy()` 方法，这时候便会调用 `stopRecording()` 方法，关闭 `mRecorder`，并用 `SharedPreferences` 保存录音文件的信息，最后将 `mRecorder` 置空，防止内存泄露
```
    public void stopRecording() {
        mRecorder.stop();
        mElapsedMillis = (System.currentTimeMillis() - mStartingTimeMillis);
        mRecorder.release();

        getSharedPreferences("sp_name_audio", MODE_PRIVATE)
                .edit()
                .putString("audio_name", mFileName)
                .putString("audio_path", mFilePath)
                .putLong("elpased", mElapsedMillis)
                .apply();
        if (mIncrementTimerTask != null) {
            mIncrementTimerTask.cancel();
            mIncrementTimerTask = null;
        }

        mRecorder = null;
    }
```

### 二、显示录音界面的 RecordAudioDialogFragment
-----
用户进行的时候，总不能让 App 跳转到另外一个界面吧，这样用户体验并不是很好，比较好的方法是显示一个对话框，让用户进行操作，既然要用对话框，必然离不开 DialogFragment，对于 DialogFragment 不是很了解，可以先看看我这篇文章 [Android 撸起袖子，自己封装 DialogFragment](http://www.jianshu.com/p/c9f20ec7277a)。

```
public class RecordAudioDialogFragment extends DialogFragment {

    private boolean mStartRecording = true;

    long timeWhenPaused = 0;

    private FloatingActionButton mFabRecord;
    private Chronometer mChronometerTime;

    public static RecordAudioDialogFragment newInstance(int maxTime) {
        RecordAudioDialogFragment dialogFragment = new RecordAudioDialogFragment();
        Bundle bundle = new Bundle();
        bundle.putInt("maxTime", maxTime);
        dialogFragment.setArguments(bundle);
        return dialogFragment;
    }

    @NonNull
    @Override
    public Dialog onCreateDialog(Bundle savedInstanceState) {
        Dialog dialog = super.onCreateDialog(savedInstanceState);
        final AlertDialog.Builder builder = new AlertDialog.Builder(getActivity());
        View view = getActivity().getLayoutInflater().inflate(R.layout.fragment_record_audio, null);

        mFabRecord.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (ContextCompat.checkSelfPermission(getActivity(), Manifest.permission.WRITE_EXTERNAL_STORAGE)
                        != PackageManager.PERMISSION_GRANTED) {
                    ActivityCompat.requestPermissions(getActivity()
                            , new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE, Manifest.permission.RECORD_AUDIO}, 1);
                }else {
                    onRecord(mStartRecording);
                    mStartRecording = !mStartRecording;
                }
            }
        });

        builder.setView(view);
        return builder.create();
    }

    private void onRecord(boolean start) {
        Intent intent = new Intent(getActivity(), RecordingService.class);
        if (start) {
            File folder = new File(Environment.getExternalStorageDirectory() + "/SoundRecorder");
            if (!folder.exists()) {
                folder.mkdir();
            }

            mChronometerTime.setBase(SystemClock.elapsedRealtime());
            mChronometerTime.start();
            getActivity().startService(intent);

        } else {
            mChronometerTime.stop();
            timeWhenPaused = 0;
            getActivity().stopService(intent);
        }
    }
}
```
可以看到在 `RecordAudioDialogFragment` 有一个 `newInstance(int maxTime)` 的静态方法供外部调用，如果想设置录音的最大时长，直接传参数进去就行了。

好的，敲黑板，重点来了，其实这个对话框的重点部分就是在 `onCreateDialog()`中，我们先加载了我们自定义的对话框的布局，当点击录音的按钮的时候，先进行相关权限的申请，这里有个巨坑，录音权限 `android.permission.RECORD_AUDIO` 在不久前还是普通权限的，不知道什么时候突然变成了危险权限，需要我们进行申请，Google 真是会玩。
```
    public Dialog onCreateDialog(Bundle savedInstanceState) {
        Dialog dialog = super.onCreateDialog(savedInstanceState);
        final AlertDialog.Builder builder = new AlertDialog.Builder(getActivity());
        View view = getActivity().getLayoutInflater().inflate(R.layout.fragment_record_audio, null);

        mFabRecord.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (ContextCompat.checkSelfPermission(getActivity(), Manifest.permission.WRITE_EXTERNAL_STORAGE)
                        != PackageManager.PERMISSION_GRANTED) {
                    ActivityCompat.requestPermissions(getActivity()
                            , new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE, Manifest.permission.RECORD_AUDIO}, 1);
                }else {
                    onRecord(mStartRecording);
                    mStartRecording = !mStartRecording;
                }
            }
        });

        builder.setView(view);
        return builder.create();
    }
```

申请好权限之后便会调用 `onRecord()` 这个方法，然后将 `boolean mStartRecording` 进行反转，这样就不用写难看的 `if else` 了，直接改变 `mStartRecording` 的值，然后在 `onRecord()` 里面进行处理

接下来看下 onRecord 干了什么
```
    private void onRecord(boolean start) {
        Intent intent = new Intent(getActivity(), RecordingService.class);
        if (mStartRecording) {
            File folder = new File(Environment.getExternalStorageDirectory() + "/SoundRecorder");
            if (!folder.exists()) {
                folder.mkdir();
            }

            mChronometerTime.setBase(SystemClock.elapsedRealtime());
            mChronometerTime.start();
            getActivity().startService(intent);

        } else {
            mChronometerTime.stop();
            timeWhenPaused = 0;
            getActivity().stopService(intent);
        }
    }
```
好吧，其实并没有干了什么大事，只是创建了保存录音文件的文件夹，然后根据 `mStartRecording` 的值进行 `RecordingService` 的启动和关闭罢了。在启动时还顺便开始了 `mChronometer` 的计时显示，这是一个 `Android` 原生的显示计时的一个控件。

### 三、播放录音的 PlaybackDialogFragment
其实，如果只是录音这一块的话，写个 `MediaPlayer` 就可以了，然而还要写播放的时间进度，以及显示一个稍微好看点的进度条，我能怎样，我也很烦啊。

外部调用这个对话框的时候，只需要传入一个包含录音文件信息的 `RecordingItem`，因为包含的信息比较多，所以最好将 `RecordingItem` 进行序列化。
```
    public static PlaybackDialogFragment newInstance(RecordingItem item) {
        PlaybackDialogFragment fragment = new PlaybackDialogFragment();
        Bundle bundle = new Bundle();
        bundle.putParcelable(ARG_ITEM, item);
        fragment.setArguments(b);
        return fragment;
    }
```

好，重点又来了，来看看 `onCreateDialog()` 方法，在加载了布局之后，给 `mSeekBar` 设置监听，`mSeekBar` 是一个显示进度条的控件，当开始播放录音时候，将录音文件的时长，设置进 `mSeekBar` 里面，播放录音的同时，运行 `mSeekBar`，通过监听 `mSeekBar` 的进度，刷新显示的播放进度。

```
    public Dialog onCreateDialog(Bundle savedInstanceState) {

        AlertDialog.Builder builder = new AlertDialog.Builder(getActivity());
        View view = getActivity().getLayoutInflater().inflate(R.layout.fragment_media_playback, null);

        mTvFileLength.setText(String.valueOf(mFileLength));
        mSeekBar.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
            @Override
            public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
                if(mMediaPlayer != null && fromUser) {
                    mMediaPlayer.seekTo(progress);
                    mHandler.removeCallbacks(mRunnable);

                    long minutes = TimeUnit.MILLISECONDS.toMinutes(mMediaPlayer.getCurrentPosition());
                    long seconds = TimeUnit.MILLISECONDS.toSeconds(mMediaPlayer.getCurrentPosition())
                            - TimeUnit.MINUTES.toSeconds(minutes);
                    mCurrentProgressTextView.setText(String.format("%02d:%02d", minutes,seconds));

                    updateSeekBar();

                } else if (mMediaPlayer == null && fromUser) {
                    prepareMediaPlayerFromPoint(progress);
                    updateSeekBar();
                }
            }

        });

        mPlayButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                onPlay(isPlaying);
                isPlaying = !isPlaying;
            }
        });

        mTvFileLength.setText(String.format("%02d:%02d", minutes,seconds));
        builder.setView(view);
        return builder.create();
    }
```
当点击播放录音的按钮之后，会调用 `onPlay()` 方法，然后根据 `isPlaying`（标识当前是否播放录音）的值，来调用不同的方法
```
    private void onPlay(boolean isPlaying){
        if (!isPlaying) {
            if(mMediaPlayer == null) {
                startPlaying(); //start from beginning
            } 
        } else {
            pausePlaying();
        }
    }
```

我们最关心的，莫过于 `startPlaying()` 这个方法，这个方法便是来开启播放录音的，我们首先将外部传入的有关的录音信息，设置给 `MediaPlayer`，然后开始调用 `mMediaPlayer.start()` 进行录音的播放，然后调用 `updateSeekbar()` 实时更新进度条的内容。当 `MediaPlayer` 的内容播放完成后，调用 `stopPlaying()` 方法，关闭 `mMediaPlayer`。
```
    private void startPlaying() {
        mMediaPlayer = new MediaPlayer();
        mMediaPlayer.setDataSource(item.getFilePath());
        mMediaPlayer.prepare();
        mSeekBar.setMax(mMediaPlayer.getDuration());

        mMediaPlayer.setOnPreparedListener(new MediaPlayer.OnPreparedListener() {
                @Override
                public void onPrepared(MediaPlayer mp) {
                    mMediaPlayer.start();
                }
            });

        mMediaPlayer.setOnCompletionListener(new MediaPlayer.OnCompletionListener() {
            @Override
            public void onCompletion(MediaPlayer mp) {
                stopPlaying();
            }
        });
        updateSeekBar();
    }
```
以上便是本文的全部内容，有关的代码我已经上传到 Github 上了，需要的 [点击这里](https://github.com/developerHaoz/SoundRecorderUtils)，喜欢的话，欢迎来波 star 和 fork

-----
### 猜你喜欢
- [Android 一起来看看知乎开源的图片选择库](http://www.jianshu.com/p/382346bf0aa9)
- [Android 能让你少走弯路的干货整理](http://www.jianshu.com/p/514656c383a2)
- [Android 撸起袖子，自己封装 DialogFragment](http://www.jianshu.com/p/c9f20ec7277a)
- [手把手教你从零开始做一个好看的 APP](http://www.jianshu.com/p/8d2d74d6046f)
