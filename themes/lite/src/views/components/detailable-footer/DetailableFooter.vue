<template>
  <div id="detailable-footer">
    <ul class="post-copyright">
      <li class="post-copyright-author">
        <strong>本文作者：</strong>
        <a href="/">欢迎访问 {{ site.author }} 的个人博客</a>
      </li>

      <li class="post-copyright-title">
        <strong>本文标题：</strong>
        <a href="#">{{ target.title }}</a>
      </li>

      <li class="post-copyright-link">
        <strong>本文链接：</strong>
        <a href="#">{{ link }}</a>
      </li>

      <li class="post-copyright-date">
        <strong>发布时间：</strong>{{ target.date|format("YYYY年M月D日 - HH时MM分") }}
      </li>

      <li class="post-copyright-license">
        <strong>版权声明： </strong>
        本博客所有文章除特别声明外，均采用<a href="https://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh" rel="license"
                            target="_blank"> 自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）</a>许可协议。转载请注明出处！
      </li>
    </ul>
  </div>
</template>

<script>
  import {Article, Page} from "@/models/article.class";
  import {Site} from "@/models/hexo-config.class";
  import {RootState} from "@/store";
  import {router} from "@/router";

  export default {
    name: "detailable-footer",
    props: {
      target: {
        required: true,
        validator: obj => obj instanceof Article || obj instanceof Page
      },
    },
    data() {
      return {
        // title: this.target.title,
        site: this.$root.$store.state.meta.hexoConfig.site,
        link: window.location.href
      }
    },
    methods: {
      showParentData() {
        console.info(this.target);
        console.info(this.site);
        console.info(this.$route.path);
        console.info(window);
      },
      getURL(url) {
        console.info(url)
        let strURL = "";
        if (url.substr(0, 7).toLowerCase() == "http://" || url.substr(0, 8).toLowerCase() == "https://") {
          strURL = url;
        } else {
          strURL = "http://" + url;
        }
        return strURL;
      }

    },
    created() {
      // this.showParentData();
    }
  }
</script>

<style lang="scss">
  @import '../../../styles/vars.scss';

  .post-copyright {
    margin: 2em 0 0;
    padding: 0.5em 1em;
    border-left: 3px solid #FF1700;
    background-color: #F9F9F9;
    list-style: none;
    text-align: left;
    line-height: 1.6rem;
  }
</style>


