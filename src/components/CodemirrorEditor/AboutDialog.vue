<template>
  <el-dialog
    title="关于"
    class="about__dialog"
    :visible="visible"
    @close="$emit('close')"
    width="30%"
    center
  >
    <div style="text-align: center">
      <h3>一款高度简洁的微信 Markdown 编辑器</h3>
      <p>扫码关注公众号 程序猿技术大咖，原创技术文章第一时间推送！</p>
      <img src="https://xcbeyond.cn/about/cxyjsdk.jpg" style="width: 40%" />
    </div>
    <template slot="footer">
      <el-button
        type="primary"
        @click="onRedirect('https://xcbeyond.cn')"
        plain
      >
        xcbeyond.cn
      </el-button>
      <el-button
        type="primary"
        @click="onRedirect('https://mp.weixin.qq.com/s/GZebuH-SDbbv-nGRtuc4AQ')"
        plain
      >
        程序猿技术大咖
      </el-button>
    </template>
  </el-dialog>
</template>

<script>
export default {
  props: {
    visible: {
      type: Boolean,
      default: false,
    },
  },
  methods: {
    onRedirect(url) {
      window.open(url)
    },
  },
}
</script>

<style lang="less" scoped>
/deep/ .el-dialog {
  min-width: 420px;
}
</style>
