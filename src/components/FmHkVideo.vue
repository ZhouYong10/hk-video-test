<template>
  <div ref="playWnd" id="playWnd" class="fm-hk-video"></div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from "vue-property-decorator";

@Component({
  name: "FmHkVideo"
})
export default class FmHkVideo extends Vue {
  @Prop({ type: String, required: true }) readonly appkey!: string; //综合安防管理平台提供的appkey，必填
  @Prop({ type: String, required: true }) readonly secret!: string; //综合安防管理平台提供的secret，必填
  @Prop({ type: String, required: true }) readonly ip!: string; //综合安防管理平台IP地址，必填
  @Prop({ type: Number, default: 443 }) readonly port?: number; //综合安防管理平台端口，若启用HTTPS协议，默认443, http协议默认为80
  @Prop({ type: Number, default: "" }) readonly sn?: string; //监控点编号
  @Prop({ type: Number, default: 0 }) readonly startTime?: number; //回放开始时间戳
  @Prop({ type: Number, default: 0 }) readonly endTime?: number; //回放结束时间戳
  @Prop({ type: Number, default: 1 }) readonly playMode?: number; //初始播放模式：0-预览，1-回放
  @Prop({ type: String, default: "D:\\SnapDir" }) readonly snapDir?: string; //抓图存储路径
  @Prop({ type: String, default: "D:\\VideoDir" }) readonly videoDir?: string; //紧急录像或录像剪辑存储路径
  /*
   * playMode指定模式的布局
   * 如未指定或指定值无效，使用默认值
   * “2x2”布局，可选值有如未指定或指定
   * 值无效，使用默认值“2x2”布局，可选
   * 值有“1x1”、“2x2”、“3x3”、“4x4”、“5x5”、
   * “1x2”、“1+2”、“1+5”、“1+7”、“1+8”、
   * “1+9” 、 “1+12” 、 “1+16” 、 “4+9” 、
   * “1+1+12”、“3+4”、“1x4”、“4x6”
   * */
  @Prop({ type: String, default: "1x1" }) readonly layout?: string;
  @Prop({ type: Number, default: 1 }) readonly enableHTTPS?: number; //是否启用HTTPS协议与综合安防管理平台交互，是为1，否为0
  @Prop({ type: String, default: "secret" }) readonly encryptedFields?: string; //加密字段，默认加密领域为secret
  @Prop({ type: Number, default: 1 }) readonly showToolbar?: number; //是否显示工具栏，0-不显示，非0-显示
  /*
   * 是否显示智能信息（如配置移动侦测后画面上的线框），0-不显示，非0-显示
   * 不指定或指定字段值为非 0，则默认显
   * 示，如指定字段值为 0，则不显示。目
   * 前支持的智能信息有智能分析信息、移
   * 动侦测信息、POS 信息后叠加信息、图
   * 片叠加信息、热成像信息、温度信息的
   * 开关配置
   * */
  @Prop({ type: Number, default: 1 }) readonly showSmart?: number;
  /*
   * 自定义工具条按钮
   * 工具条分为上下两个工具条，
   * 上工具条包括监控点名称控件和关闭
   * 按钮；预览下工具条按钮有声音、抓图、
   * 云台控制、3D 放大、电子放大、语音
   * 对讲、显示监控点信息、主子码流切换、
   * 紧急录像、鹰眼和即时回放按钮；
   * 自定义按钮定义（括号内为对应 10 进
   * 制 ID 值）： 0x0000(0)- 监 控 点 名 称 按 钮
   * 0x0010(16)-关闭按钮
   * 0x0100(256)- 预 览 回 放 声 音
   * 0x0101(257)- 预 览 回 放 抓 图
   * 0x0102(258)- 预 览 回 放 电 子 放 大
   * 0x0103(259)-预览回放显示监控点信息
   * 0x0104(260)-小鹰眼 0x0200(512)-预览
   * 云台控制 0x0201(513)-预览 3D 放大
   * 0x0202(514)- 预 览 语 音 对 讲
   * 0x0203(515)- 预 览 主 子 码 流 切 换
   * 0x0204(516)- 预 览 紧 急 录 像
   * 0x205(517)- 预 览 即 时 回 放
   * 0x0300(768)- 回 放 录 像 剪 辑
   * 0x0301(769)-回放录像下载
   * 下工具条按指定按钮字符串中按钮 ID
   * 顺序来显示
   * 指定为空字符串时不显示工具条；指定
   * 重复、超出可选范围值、非法值将返回
   * 失败；指定功能不支持的按钮 ID 将不
   * 显示该按钮（如预览模式指定下载按
   * 钮），如去除不支持的按钮 ID 后无其它
   * 指定按钮 ID 时，工具条不显示
   * */
  @Prop({
    type: String,
    default: "0,16,256,257,258,259,260,512,513,514,515,516,517,768,769"
  })
  readonly buttonIDs?: string;
  width = 1000;
  height = 600;
  oWebControl = null;
  initCount = 0;
  pubKey = "";

  mounted() {
    const $playWnd: any = this.$refs.playWnd;
    this.width = $playWnd.clientWidth;
    this.height = $playWnd.clientHeight;
    // 监听resize事件，使插件窗口尺寸跟随DIV窗口变化
    // eslint-disable-next-line no-undef
    $(window).resize(() => {
      if (this.oWebControl != null) {
        this.oWebControl.JS_Resize(this.width, this.height);
        this.setWndCover();
      }
    });

    // 监听滚动条scroll事件，使插件窗口跟随浏览器滚动而移动
    // eslint-disable-next-line no-undef
    $(window).scroll(() => {
      if (this.oWebControl != null) {
        this.oWebControl.JS_Resize(this.width, this.height);
        this.setWndCover();
      }
    });
    console.log(this.width, this.height, " 0000000000000000000000000000000");
    this.initPlugin();
  }

  destroyed() {
    if (this.oWebControl != null) {
      this.oWebControl.JS_HideWnd(); // 先让窗口隐藏，规避可能的插件窗口滞后于浏览器消失问题
      this.oWebControl.JS_Disconnect().then(
        () => {
          // 断开与插件服务连接成功
        },
        () => {
          // 断开与插件服务连接失败
        }
      );
    }
  }

  // 创建播放实例
  initPlugin() {
    // eslint-disable-next-line no-undef
    this.oWebControl = new WebControl({
      szPluginContainer: "playWnd", // 指定容器id
      iServicePortStart: 15900, // 指定起止端口号，建议使用该值
      iServicePortEnd: 15909,
      szClassId: "23BF3B0A-2C56-4D97-9C03-0CB103AA8F11", // 用于IE10使用ActiveX的clsid
      cbConnectSuccess: () => {
        // 创建WebControl实例成功
        this.oWebControl
          .JS_StartService("window", {
            // WebControl实例创建成功后需要启动服务
            dllPath: "./VideoPluginConnect.dll" // 值"./VideoPluginConnect.dll"写死
          })
          .then(
            () => {
              // 启动插件服务成功
              this.oWebControl.JS_SetWindowControlCallback({
                // 设置消息回调
                cbIntegrationCallBack: this.cbIntegrationCallBack
              });

              this.oWebControl
                .JS_CreateWnd("playWnd", this.width, this.height)
                .then(() => {
                  //JS_CreateWnd创建视频播放窗口，宽高可设定
                  this.init(); // 创建播放实例成功后初始化
                });
            },
            () => {
              // 启动插件服务失败
            }
          );
      },
      cbConnectError: () => {
        // 创建WebControl实例失败
        this.oWebControl = null;
        // eslint-disable-next-line no-undef
        $("#playWnd").html("插件未启动，正在尝试启动，请稍候...");
        // eslint-disable-next-line no-undef
        WebControl.JS_WakeUp("VideoWebPlugin://"); // 程序未启动时执行error函数，采用wakeup来启动程序
        this.initCount++;
        if (this.initCount < 3) {
          setTimeout(() => {
            this.initPlugin();
          }, 3000);
        } else {
          // eslint-disable-next-line no-undef
          $("#playWnd").html("插件启动失败，请检查插件是否安装！");
        }
      },
      cbConnectClose: bNormalClose => {
        // 异常断开：bNormalClose = false
        // JS_Disconnect正常断开：bNormalClose = true
        console.log(bNormalClose, "cbConnectClose");
        this.oWebControl = null;
      }
    });
  }

  // 设置窗口控制回调
  setCallbacks() {
    this.oWebControl.JS_SetWindowControlCallback({
      cbIntegrationCallBack: this.cbIntegrationCallBack
    });
  }

  // 推送消息
  cbIntegrationCallBack(oData) {
    console.log(JSON.stringify(oData.responseMsg), " 推送消息===============");
  }

  //初始化
  init() {
    this.getPubKey(() => {
      this.oWebControl
        .JS_RequestInterface({
          funcName: "init",
          argument: JSON.stringify({
            appkey: this.appkey, //API网关提供的appkey
            secret: this.secret, //API网关提供的secret
            ip: this.ip, //API网关IP地址
            playMode: this.playMode, //播放模式（决定显示预览还是回放界面）
            port: this.port, //端口
            snapDir: this.snapDir, //抓图存储路径
            videoDir: this.videoDir, //紧急录像或录像剪辑存储路径
            layout: this.layout, //布局
            enableHTTPS: this.enableHTTPS, //是否启用HTTPS协议
            encryptedFields: this.encryptedFields, //加密字段
            showToolbar: this.showToolbar, //是否显示工具栏
            showSmart: this.showSmart, //是否显示智能信息
            buttonIDs: this.buttonIDs //自定义工具条按钮
          })
        })
        .then(oData => {
          console.log(
            oData,
            " init JS_RequestInterface ============================"
          );
          this.oWebControl.JS_Resize(this.width, this.height); // 初始化后resize一次，规避firefox下首次显示窗口后插件窗口未与DIV窗口重合问题
          if (this.sn) {
            if (this.playMode) {
              this.playback(this.sn, this.startTime, this.endTime);
            } else {
              this.preview(this.sn);
            }
          }
        });
    });
  }

  // 实时预览
  preview(sn, streamMode = 0, transMode = 1, gpuMode = 0, wndId = -1) {
    this.oWebControl.JS_RequestInterface({
      funcName: "startPreview",
      argument: JSON.stringify({
        cameraIndexCode: sn, //监控点编号
        streamMode: streamMode, //主子码流标识
        transMode: transMode, //传输协议
        gpuMode: gpuMode, //是否开启GPU硬解
        wndId: wndId //可指定播放窗口
      })
    });
  }

  // 回放
  playback(
    sn,
    startTimeStamp,
    endTimeStamp,
    recordLocation = 0,
    transMode = 1,
    gpuMode = 0,
    wndId = -1
  ) {
    this.oWebControl.JS_RequestInterface({
      funcName: "startPlayback",
      argument: JSON.stringify({
        cameraIndexCode: sn, //监控点编号
        startTimeStamp: Math.floor(startTimeStamp / 1000).toString(), //录像查询开始时间戳，单位：秒
        endTimeStamp: Math.floor(endTimeStamp / 1000).toString(), //录像结束开始时间戳，单位：秒
        recordLocation: recordLocation, //录像存储类型：0-中心存储，1-设备存储
        transMode: transMode, //传输协议：0-UDP，1-TCP
        gpuMode: gpuMode, //是否启用GPU硬解，0-不启用，1-启用
        wndId: wndId //可指定播放窗口
      })
    });
  }

  //获取公钥
  getPubKey(callback) {
    this.oWebControl
      .JS_RequestInterface({
        funcName: "getRSAPubKey",
        argument: JSON.stringify({
          keyLength: 1024
        })
      })
      .then(oData => {
        console.log(oData);
        if (oData.responseMsg.data) {
          this.pubKey = oData.responseMsg.data;
          callback();
        }
      });
  }

  //RSA加密
  setEncrypt(value) {
    // eslint-disable-next-line no-undef
    const encrypt = new JSEncrypt();
    encrypt.setPublicKey(this.pubKey);
    return encrypt.encrypt(value);
  }
  // 设置窗口裁剪，当因滚动条滚动导致窗口需要被遮住的情况下需要JS_CuttingPartWindow部分窗口
  setWndCover() {
    // eslint-disable-next-line no-undef
    const iWidth = $(window).width();
    // eslint-disable-next-line no-undef
    const iHeight = $(window).height();
    // eslint-disable-next-line no-undef
    const oDivRect = $("#playWnd")
      .get(0)
      .getBoundingClientRect();

    let iCoverLeft = oDivRect.left < 0 ? Math.abs(oDivRect.left) : 0;
    let iCoverTop = oDivRect.top < 0 ? Math.abs(oDivRect.top) : 0;
    let iCoverRight =
      oDivRect.right - iWidth > 0 ? Math.round(oDivRect.right - iWidth) : 0;
    let iCoverBottom =
      oDivRect.bottom - iHeight > 0 ? Math.round(oDivRect.bottom - iHeight) : 0;

    iCoverLeft = iCoverLeft > 1000 ? 1000 : iCoverLeft;
    iCoverTop = iCoverTop > 600 ? 600 : iCoverTop;
    iCoverRight = iCoverRight > 1000 ? 1000 : iCoverRight;
    iCoverBottom = iCoverBottom > 600 ? 600 : iCoverBottom;

    this.oWebControl.JS_RepairPartWindow(0, 0, 1001, 600); // 多1个像素点防止还原后边界缺失一个像素条
    if (iCoverLeft != 0) {
      this.oWebControl.JS_CuttingPartWindow(0, 0, iCoverLeft, 600);
    }
    if (iCoverTop != 0) {
      this.oWebControl.JS_CuttingPartWindow(0, 0, 1001, iCoverTop); // 多剪掉一个像素条，防止出现剪掉一部分窗口后出现一个像素条
    }
    if (iCoverRight != 0) {
      this.oWebControl.JS_CuttingPartWindow(
        1000 - iCoverRight,
        0,
        iCoverRight,
        600
      );
    }
    if (iCoverBottom != 0) {
      this.oWebControl.JS_CuttingPartWindow(
        0,
        600 - iCoverBottom,
        1000,
        iCoverBottom
      );
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="stylus"></style>
