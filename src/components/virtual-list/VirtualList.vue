<template>
  <div class="list">
    <VirtualCollection
      :cell-size-and-position-getter="cellSizeAndPositionGetter"
      :collection="list"
      :identifier="identifier"
      :width="width"
      :height="height"
      @infinite="infinite"
    >
      <slot />
      <template v-slot:cell="props">
        <Item :column="props.data" @handleLike="handleLike" />
      </template>
    </VirtualCollection>
  </div>
</template>

<script>
import dayjs from 'dayjs';
import { mapGetters } from 'vuex';
import VirtualCollection from '@/components/collect/VirtualCollection';
import Alert from '@/components/alert';
import throttle from 'lodash/throttle';
import Item from './Item';
import { randomColor, replaceBigImg, replaceSmallImg } from '@/util';
import { getClient } from '@/util/dom';
const columnWidth = 200; // 屏幕小于200则1列

export default {
  components: {
    VirtualCollection,
    Item
  },
  props: {
    list: {
      type: Array,
      default() {
        return [];
      }
    },
    identifier: {
      type: Number,
      default: +new Date()
    }
  },
  data() {
    return {
      scrollY: 0,
      columnHeight: [],
      column: 0,
      width: getClient().width,
      height: getClient().height
    };
  },
  computed: {
    ...mapGetters(['user', 'likeStatus', 'showTab']),
    listMap() {
      const map = new Map();
      for (const item of this.list) {
        map.set(item.id, item);
      }
      return map;
    }
  },
  watch: {
    list: {
      handler(val, old) {
        if (val.length === 0) {
          this.columnHeight = new Array(this.column).fill(0);
        } else {
          const list = val.filter(e => !old.includes(e));
          this.handleList(list);
        }
      }
    },
    likeStatus(val, old) {
      // 注意 List不一定找得到item 要判断下
      const { illustId, like } = val;
      const item = this.listMap.get(illustId);
      // this.$set(item, 'isLiked', like);
      if (item) {
        this.$set(item, 'isLiked', like);
      }
    }
  },
  mounted() {
    this.waterFall();
    window.addEventListener('resize', throttle(this.waterFall));
  },
  destroyed() {
    window.removeEventListener('resize', this.waterFall);
  },
  methods: {
    infinite($state) {
      this.$emit('infinite', $state);
    },
    cellSizeAndPositionGetter(item, index) {
      return {
        width: item.width,
        height: item.height,
        x: item.x,
        y: item.y
      };
    },
    handleLike(data) {
      if (!this.user.id) {
        this.$router.push({
          name: 'Login',
          query: {
            return_to: window.location.href
          }
        });
        return;
      }
      const item = this.listMap.get(data.id);
      const flag = item.isLiked;
      const params = {
        userId: this.user.id,
        illustId: data.id
      };
      if (!flag) {
        this.$set(item, 'isLiked', true); // 先强制视图更新 防止网络延迟不动
        this.$store.dispatch('handleCollectIllust', params)
          .then(() => {})
          .catch(() => {
            this.$set(item, 'isLiked', false); // 失败的话在改回去
            Alert({
              content: '收藏失败'
            });
          });
      } else {
        this.$set(item, 'isLiked', false);
        this.$store.dispatch('deleteCollectIllust', params)
          .then(() => {})
          .catch(() => {
            this.$set(item, 'isLiked', true);
            Alert({
              content: '取消收藏失败'
            });
          });
      }
    },
    waterFall() {
      this.width = getClient().width;
      this.height = getClient().height;
      const column = parseInt(localStorage.getItem('waterfull-column'));
      if (column) {
        this.column = column;
      } else {
        this.column = Math.ceil(this.width / columnWidth);
      }
      this.columnHeight = new Array(this.column).fill(0);
      this.handleList(this.list);
    },
    handleList(list) {
      for (let i = 0; i < list.length; i++) {
        const tmp = list[i];
        const per = tmp.height / tmp.width;
        const width = Math.floor((this.width - 16) / this.column);
        const height = Math.max(Math.min(width * per, 400), 100);
        // 找出最小列
        let minHeight = this.columnHeight[0];
        let index = 0;
        for (let j = 0; j < this.columnHeight.length; j++) {
          if (minHeight > this.columnHeight[j]) {
            minHeight = this.columnHeight[j];
            index = j;
          }
        }
        tmp.x = index * width;
        tmp.y = this.columnHeight[index];
        this.columnHeight[index] += height;

        tmp['height'] = height;
        tmp['width'] = width;
        tmp['src'] = replaceSmallImg(tmp.imageUrls[0].medium);
        tmp['setu'] = !!((tmp.xrestrict === 1 || tmp.sanityLevel > 5)) && this.user.username !== 'pixivic';
        tmp['style'] = {
          backgroundColor: randomColor()
        };
        tmp['itemHeight'] = parseInt(per * this.width);
        tmp['avatarSrc'] = replaceBigImg(tmp.artistPreView.avatar);
        tmp['createDate'] = dayjs(tmp.createDate).format('YYYY-MM-DD');
        tmp['imgs'] = tmp.imageUrls.reduce((pre, cur) => {
          return pre.concat(replaceBigImg(cur.original));
        }, []);
        tmp['originalSrc'] = replaceBigImg(tmp.imageUrls[0].original);
      }
    }
  }
};
</script>

<style lang="stylus" scope>
.list
  position relative
  background-color #fff
</style>
