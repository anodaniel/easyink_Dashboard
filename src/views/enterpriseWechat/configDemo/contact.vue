<script>
import { ENTERPRISE_WECHAT_AUTH_STEP_TEXT, SERVER_TYPE_THIRD } from '@/utils/constant';
import { replaceUrlProtocol } from '@/utils/common';
const LAST_STEP_NUMBER = 3;
export default {
  components: {},
  props: {
    defaultDomain: {
      type: Object,
      default: () => {}
    }
  },
  data() {
    return {
      active: 0,
      STEP_TEXT: ENTERPRISE_WECHAT_AUTH_STEP_TEXT,
      // 是否为第三方应用
      isThirdType: this.$store.state.serverInfo && this.$store.state.serverInfo.serverType === SERVER_TYPE_THIRD
    };
  },
  computed: {},
  watch: {},
  created() {},
  mounted() {},
  methods: {
    next() {
      if (this.active !== LAST_STEP_NUMBER) {
        this.active = this.active + 1;
      } else {
        this.$emit('closeDrawer');
      }
    },
    previousStep() {
      if (this.active-- < 0) this.active = LAST_STEP_NUMBER;
    },
    replaceUrl(url) {
      return replaceUrlProtocol(url);
    }
  }
};
</script>

<template>
  <div class="contact-demo enterprise-wechat-demo">
    <div>
      <el-steps class="step-div" :active="active" align-center>
        <el-step title="获取企业ID" description="登录企业微信后台" />
        <el-step title="创建应用" description="获取应用ID和应用Secret" />
        <el-step title="获取通讯录Secret" description="完成通讯录配置" />
        <el-step title="授权登录" description="员工扫码登录系统" />
      </el-steps>
    </div>
    <div v-if="active === 0" class="content-div">
      <div class="part">
        <div class="content-title">
          企业管理员扫码登录
          <el-link type="primary" href="https://work.weixin.qq.com/wework_admin/loginpage_wx" target="_blank">企业微信后台</el-link>，进入
          <el-link type="primary" href="https://work.weixin.qq.com/wework_admin/frame#profile" target="_blank">【我的企业】</el-link>
          -【企业信息】，复制企业ID。</div>
        <el-image
          class="part-img"
          :src="require('@/assets/example/demo/contact-step1-1.jpg')"
          :preview-src-list="[require('@/assets/example/demo/contact-step1-1.jpg')]"
        />
      </div>
    </div>
    <div v-if="active === 1" class="content-div">
      <div class="part">
        <div class="content-title">（1）进入
          <el-link type="primary" href="https://work.weixin.qq.com/wework_admin/frame#apps" target="_blank">【应用管理】</el-link>-【创建应用】。
        </div>
        <el-image
          class="part-img"
          :src="require('@/assets/example/demo/contact-step2-1.png')"
          :preview-src-list="[require('@/assets/example/demo/contact-step2-1.png')]"
        />
      </div>
      <div class="part">
        <div class="content-title">（2）将以下信息填写到应用中，并设置该应用有哪些成员可见。</div>
        <div class="content-text">应用名称：联络易</div>
        <div class="content-text" style="display: flex; align-items: center;">
          应用logo：
          <el-image
            style="width: 42px"
            :src="require('@/assets/logo/easyWeComLogo.png')"
          />
        </div>
        <el-image
          class="part-img"
          :src="require('@/assets/example/demo/contact-step2-2.jpg')"
          :preview-src-list="[require('@/assets/example/demo/contact-step2-2.jpg')]"
        />
      </div>
      <div class="part">
        <div class="content-title">（3）获取应用信息。</div>
        <div class="content-text">保持应用为开启状态，复制应用ID，点击Secret旁“查看”按钮，即可收到“企业微信团队”发出的消息，查看并复制应用Secret。</div>
        <el-image
          class="part-img"
          :src="require('@/assets/example/demo/contact-step2-3.jpg')"
          :preview-src-list="[require('@/assets/example/demo/contact-step2-3.jpg')]"
        />
      </div>
      <div class="part">
        <div class="content-title">（4）添加企业IP白名单。</div>
        <div class="content-text"> 找到"企业可信IP"，点击“配置”按钮，填写当前系统的IP地址。</div>
        <el-image
          class="part-img"
          :src="require('@/assets/example/demo/contact-step2-4.jpg')"
          :preview-src-list="[require('@/assets/example/demo/contact-step2-4.jpg')]"
        />
      </div>
    </div>
    <div v-if="active === 2" class="content-div">
      <div class="part">
        <div class="content-title">
          （1）进入
          <el-link type="primary" href="https://work.weixin.qq.com/wework_admin/frame#manageTools" target="_blank">【管理工具】</el-link>
          -【通讯录同步】，点击“开启API接口同步”，进入（2）
        </div>
        <el-image
          class="part-img"
          :src="require('@/assets/example/demo/contact-step3-1.jpg')"
          :preview-src-list="[require('@/assets/example/demo/contact-step3-1.jpg')]"
        />
      </div>
      <div class="part">
        <div class="content-title">
          （2）获取通讯录Secret。
        </div>
        <div class="content-text">将权限改为API编辑通讯录并保存，可在EasyInk管理员工信息；点击Secret旁“查看”按钮，即可收到“企业微信团队”发出的消息，查看并复制通讯录Secret。</div>
        <el-image
          class="part-img"
          :src="require('@/assets/example/demo/contact-step3-2.jpg')"
          :preview-src-list="[require('@/assets/example/demo/contact-step3-2.jpg')]"
        />
      </div>
      <div class="part">
        <div class="content-title">
          （3）添加企业IP白名单
        </div>
        <div class="content-text">点击"企业可信IP"旁边的配置按钮，<span style="color: red;">当前弹窗先不要关闭</span>，此时企微客户端将收到一条企业微信团队发送的确认操作。</div>
        <el-image
          class="part-img"
          :src="require('@/assets/example/demo/contact-step3-3.jpg')"
          :preview-src-list="[require('@/assets/example/demo/contact-step3-3.jpg')]"
        />
      </div>
      <div class="part">
        <div class="content-title">
          （4）在客户端确认操作后，企微后台弹窗即转为可编辑的输入框，填进当前系统的IP地址后，点击"确定"按钮保存即可。
        </div>
        <div class="content-text" />
        <el-image
          class="part-img"
          :src="require('@/assets/example/demo/contact-step3-4.jpg')"
          :preview-src-list="[require('@/assets/example/demo/contact-step3-4.jpg')]"
        />
      </div>
    </div>
    <div v-if="active === 3" class="content-div">
      <div class="part">
        <div v-if="isThirdType" class="content-title">
          （1）域名备案：{{ replaceUrl(defaultDomain.dashboard) }}
        </div>
        <div v-else class="content-title">
          （1）将WeComDashboard模块进行域名备案，如www.easyink.net:8091。
        </div>
      </div>
      <div class="part">
        <div class="content-title">
          （2）在企业微信后台
          <el-link type="primary" href="https://work.weixin.qq.com/wework_admin/frame#apps" target="_blank">【应用管理】</el-link>
          找到联络易应用，找到“企业微信授权登录”，进入页面。
        </div>
        <el-image
          class="part-img"
          :src="require('@/assets/example/demo/contact-step4-1.jpg')"
          :preview-src-list="[require('@/assets/example/demo/contact-step4-1.jpg')]"
        />
      </div>
      <div class="part">
        <div class="content-title">
          （3）找到Web网页，编辑授权回调域，将（1）中备案好的域名填入并保存，即可实现员工扫码登录EasyInk。
        </div>
        <el-image
          class="part-img"
          :src="require('@/assets/example/demo/contact-step4-2.jpg')"
          :preview-src-list="[require('@/assets/example/demo/contact-step4-2.jpg')]"
        />
      </div>
    </div>
    <div class="btn-div">
      <el-button v-if="active != 0" style="margin-top: 12px;" @click="previousStep">上一步</el-button>
      <el-button type="primary" style="margin-top: 12px;" @click="next">{{ STEP_TEXT[active] }}</el-button>
    </div>
  </div>
</template>
<style lang="scss" src="./styles/common.scss" scoped>
</style>
