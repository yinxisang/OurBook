<template>
  <div class="search-box">
    <input
      @input="query = $event.target.value"
      aria-label="Search"
      :value="query"
      autocomplete="off"
      spellcheck="false"
      @focus="focused = true"
      @blur="focused = false"
      @keyup.enter="go(focusIndex)"
      @keyup.up="onUp"
      @keyup.down="onDown">
    <ul class="suggestions"
      v-if="showSuggestions"
      :class="{ 'align-right': alignRight }"
      @mouseleave="unfocus">
      <li class="suggestion" v-for="(s, i) in suggestions"
        :class="{ focused: i === focusIndex }"
        @mousedown="goDetail(i)"
        @mouseenter="focus(i)">
        <a :href="s.path" @click.prevent>
          <span v-if="s.univ">{{s.univ}} &gt;&nbsp;</span>
          <span class="page-title">{{ s.title || s.path }}</span>
          <span v-if="s.header  && s.header.title " class="header">&gt; {{ s.header.title }}</span>
        </a>
      </li>
    </ul>
  </div>
</template>

<script>
import univ from 'constants/univ.js';

export default {
  data () {
    return {
      query: '',
      focused: false,
      focusIndex: 0
    }
  },
  computed: {
    showSuggestions () {
      return (
        this.focused &&
        this.suggestions &&
        this.suggestions.length
      )
    },
    suggestions () {
      const query = this.query.trim().toLowerCase()
      if (!query) {
        return
      }

      const { pages, themeConfig } = this.$site
      // 最大显示建议数
      const max = themeConfig.searchMaxSuggestions || 5
      // 筛选数组
      let res = []

      const localePath = this.$localePath
      // input框中待输入的关键词处理
      const qlist = query.replace(' ', '').split('')
      // custom search logic

      // const matches = (item) => {
      //   if(!item) {
      //     return false
      //   }
      //   // return qlist.every((v) => {
      //   //   return item.indexOf(v) > -1
      //   // })
      //   let order = 0
      //   qlist.forEach(key => {
      //     if (item.indexOf(key) != -1) {
      //       order++
      //     }
      //   })
      //   if (order >= qlist.length) {
      //     priorityList.push(Math.round((order / item.length) * 100))
      //     return true
      //   } else {
      //     return false
      //   }
      // }

      // 检索
      const search = (str) => {
        const match = qlist.filter(key => str.indexOf(key) !== -1)
        return match.length
      }
      // 加权
      const newMatch = (item, p) => {
        let order = 0
        let total = 0
        let count = 0
        // 匹配完整
        let defaultStr = `${item.name}${item.pTitle}${(item.h && item.h.title) ? item.h.title : ''}`
        let aim = qlist.every(key => defaultStr.indexOf(key) !== -1)
        if (!aim) {
          return
        }


        // 如果在大学名称中 权重为9
        if (item.name) {
          order = search(item.name)
          total += order
          count += order * 5
        }
        
        // 如果在主页面标题中 权重为5
        if (item.pTitle) {
          order = search(item.pTitle)
          total += order
          count += order * 3
        }

        // 如果在主页面副标题中 权重为3
        if (item.h && item.h.title) {
          order = search(item.h.title)
          total += order
          count += order * 1
        }

        let np = Object.create(null)
        if (total >= qlist.length) {
          if (item.h) {
            np = Object.assign({}, p, {
              path: `${p.path}#${item.h.slug}`,
              header: item.h,
              univ: name ? name : '',
              count
            })
          } else {
            if (item.name) {
              np = Object.assign({}, p, {
                title: item.name,
                header: {
                  title: p.title
                },
                count
              });
            }
          }
          res.push(np)
        }
      }

      for (let i = 0; i < pages.length; i++) {
        // 超过搜索限制停止
        // if (res.length >= max) break
        const p = pages[i]
        // filter out results that do not match current locale
        if (this.getPageLocalePath(p) !== localePath) {
          continue
        }
        // 如果文章内存在小标题，那么
        const un = /\/(.*)\//g.exec(p.path)
        const name = un ? univ[un[1]] : ''
        if(p.headers) {

          p.headers.forEach((h) => {
            const t = p.title

            newMatch({name, pTitle: p.title, h}, p)

          });
        } else {

          newMatch({name, pTitle: p.title}, p)

        }
      }
      

      res = res.sort((a, b) => {
        return b.count - a.count
      })

      console.log(res.slice(0, max))

      return res.slice(0, max)
    },
    // make suggestions align right when there are not enough items
    alignRight () {
      const navCount = (this.$site.themeConfig.nav || []).length
      const repo = this.$site.repo ? 1 : 0
      return navCount + repo <= 2
    }
  },
  methods: {
    getPageLocalePath (page) {
      for (const localePath in this.$site.locales || {}) {
        if (localePath !== '/' && page.path.indexOf(localePath) === 0) {
          return localePath
        }
      }
      return '/'
    },
    onUp () {
      if (this.showSuggestions) {
        if (this.focusIndex > 0) {
          this.focusIndex--
        } else {
          this.focusIndex = this.suggestions.length - 1
        }
      }
    },
    onDown () {
      if (this.showSuggestions) {
        if (this.focusIndex < this.suggestions.length - 1) {
          this.focusIndex++
        } else {
          this.focusIndex = 0
        }
      }
    },
    // 不能使用保留字 go
    goDetail (i) {
      if(!this.showSuggestions) {
        return;
      }
      this.$router.push(this.suggestions[i].path)
      this.query = ''
      this.focusIndex = 0
    },
    focus (i) {
      this.focusIndex = i
    },
    unfocus () {
      this.focusIndex = -1
    }
  }
}
</script>

<style lang="stylus">
@import './styles/config.styl'

.search-box
  display inline-block
  position relative
  margin-right 0.5rem
  input
    cursor pointer
    width 10rem
    color lighten($textColor, 25%)
    display inline-block
    border 1px solid darken($borderColor, 10%)
    border-radius 2rem
    font-size 0.9rem
    line-height 2rem
    padding 0 0.5rem 0 2rem
    outline none
    transition all .2s ease
    background #fff url(./search.svg) 0.6rem 0.5rem no-repeat
    background-size 1rem
    &:focus
      cursor auto
      border-color $accentColor
  .suggestions
    background #fff
    width 20rem
    // max-height 20rem
    // overflow-y auto
    position absolute
    top 1.5rem
    border 1px solid darken($borderColor, 10%)
    border-radius 6px
    padding 0.4rem
    list-style-type none
    &.align-right
      right 0
  .suggestion
    line-height 1.4
    padding 0.4rem 0.6rem
    border-radius 4px
    a
      color lighten($textColor, 35%)
      .page-title
        font-weight 600
      .header
        font-size 0.9em
        margin-left 0.25em
    &.focused
      background-color #f3f4f5
      a
        color $accentColor

@media (max-width: $MQNarrow)
  .search-box input
    width 0
    border-color transparent
    position relative
    left 1rem
    &:focus
      left 0
      width 10rem

@media (max-width: $MQMobile)
  .search-box
    margin-right 0
    .suggestions
      right 0

@media (max-width: $MQMobileNarrow)
  .search-box
    .suggestions
      width calc(100vw - 4rem)
    input:focus
      width 8rem
</style>
