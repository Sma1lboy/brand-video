<img src="https://raw.githubusercontent.com/Sma1lboy/brand-video/main/assets/banner.svg" width="100%" alt="brand-video">

# brand-video

brand-studio 的通用 video producer(instruction-grade skill):脚本/参考视频分析层(storyboard + 相机权重)、per-project Remotion 生产层、TTS/自声克隆/bgm/facecam 插槽。

挂载方式:作为 git submodule 放在 brand-studio 仓库的 `skills/brand-studio/producers/brand-video-video`,宿主 repo 经 brand-studio 的 metadata(`skills.video: brand-video`)绑定使用。Remotion 工具链不在本仓库——在宿主的 `.studio/out/<name>/` 每个视频项目里物化。
