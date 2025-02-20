<template>
  <div class="home-container">
    <div class="header">
      <user-info
        class="header-item user-info"
        :user-id="userId"
        :user-name="userName"
        :user-avatar="userAvatar"
        @log-out="handleLogOut"
      ></user-info>
      <language-icon class="header-item language"></language-icon>
      <switch-theme class="header-item theme"></switch-theme>
    </div>
    <stream-preview v-if="!isMobile" ref="streamPreviewRef"></stream-preview>
    <room-control
      ref="roomControlRef"
      :given-room-id="givenRoomId"
      :user-name="userName || userId"
      @create-room="handleCreateRoom"
      @enter-room="handleEnterRoom"
    ></room-control>
  </div>
</template>

<script>
import UserInfo from '@/TUIRoom/components/RoomHeader/UserInfo';
import LanguageIcon from '@/TUIRoom/components/base/Language.vue';
import SwitchTheme from '@/TUIRoom/components/base/SwitchTheme.vue';
import StreamPreview from '@/TUIRoom/components/RoomHome/StreamPreview.vue';
import RoomControl from '@/TUIRoom/components/RoomHome/RoomControl';
import { getBasicInfo } from '@/config/basic-info-config';
import TUIRoomEngine from '@tencentcloud/tuiroom-engine-electron';
import useGetRoomEngine from '@/TUIRoom/hooks/useRoomEngine';
import { isMobile } from '../TUIRoom/utils/useMediaValue';

const roomEngine = useGetRoomEngine();
export default {
  name: 'Home',
  components: {
    UserInfo,
    LanguageIcon,
    StreamPreview,
    RoomControl,
    SwitchTheme,
  },
  data() {
    return {
      givenRoomId: '',
      basicInfo: null,
      userName: '',
      userAvatar: '',
      userId: '',
      isMobile,
    };
  },
  async mounted() {
    sessionStorage.removeItem('tuiRoom-roomInfo');
    sessionStorage.removeItem('tuiRoom-userInfo');
    this.givenRoomId = this.$route.query.roomId || '';

    if (sessionStorage.getItem('tuiRoom-userInfo')) {
      this.basicInfo = JSON.parse(sessionStorage.getItem('tuiRoom-userInfo'));
    } else {
      this.basicInfo = await getBasicInfo();
      this.basicInfo && sessionStorage.setItem('tuiRoom-userInfo', JSON.stringify(this.basicInfo));
    }
    this.userName = this.basicInfo.userName;
    this.userAvatar = this.basicInfo.userAvatar;
    this.userId = this.basicInfo.userId;
    const { sdkAppId, userId, userSig } = this.basicInfo;
    // 登录 TUIRoomEngine
    await TUIRoomEngine.login({ sdkAppId, userId, userSig });
    if (!this.isMobile) {
      this.$refs.streamPreviewRef.startStreamPreview();
    }
  },
  methods: {
    setTUIRoomData(action, mode) {
      const roomParam = this.isMobile
        ? this.$refs.roomControlRef.getRoomParam() : this.$refs.streamPreviewRef.getRoomParam();
      const roomData = {
        action,
        roomMode: mode || 'FreeToSpeak',
        roomParam,
      };
      sessionStorage.setItem('tuiRoom-roomInfo', JSON.stringify(roomData));
    },
    async checkRoomExistWhenCreateRoom(roomId) {
      let isRoomExist = false;
      const tim = roomEngine.instance?.getTIM();
      try {
        await tim.searchGroupByID(roomId);
        isRoomExist = true;
      } catch (error) {
        // 房间不存在
      }
      return isRoomExist;
    },
    // 创建房间时生成房间号
    async generateRoomId() {
      const roomId = Math.ceil(Math.random() * 1000000);
      const isRoomExist = await this.checkRoomExistWhenCreateRoom(String(roomId));
      if (isRoomExist) {
        return await this.generateRoomId();
      }
      return roomId;
    },
    // 处理点击【创建房间】
    async handleCreateRoom(mode) {
      this.setTUIRoomData('createRoom', mode);
      const roomId = await this.generateRoomId();
      this.$router.push({ path: 'room', query: { roomId } });
    },
    // 处理点击【进入房间】
    async handleEnterRoom(roomId) {
      this.setTUIRoomData('enterRoom');
      this.$router.push({
        path: 'room',
        query: {
          roomId,
        },
      });
    },
    // 处理用户点击页面左上角【退出登录】
    handleLogOut() {
      // 接入方处理 logout 方法
    },
  },
};
</script>

<style>
@import '../TUIRoom/assets/style/black-theme.scss';
@import '../TUIRoom/assets/style/white-theme.scss';

</style>

<style lang="scss" scoped>
.home-container {
  width: 100%;
  height: 100%;
  background: var(--background-color-style);
  color: #B3B8C8;
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: PingFangSC-Medium;
  transition: background .5s,color .5s;
  :not([class|="el"]) {
    transition: background-color .5s,color .5s;
  }
  .header {
    width: 100%;
    position: absolute;
    top: 0;
    padding: 22px 24px;
    display: flex;
    align-items: center;
    .header-item {
      &:not(:first-child) {
        margin-left: 16px;
      }
      .language{
        cursor: pointer;
      }
      .theme{
        cursor: pointer;
      }
    }
  }
}
</style>
