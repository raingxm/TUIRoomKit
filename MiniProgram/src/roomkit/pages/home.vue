<template>
  <div :class="[isMobile? 'home-container-H5' : 'home-container']" data-theme="white" class="white-theme">
    <div :class="[isMobile ? 'header-H5' : 'header']">
      <user-info
        v-if="!isMobile"
        :user-id="userId"
        :user-name="userName"
        :avatar-url="avatarUrl"
        @log-out="handleLogOut"
      ></user-info>
      <div v-if="!isMobile" class="container-icon">
        <language-icon class="header-item language"></language-icon>
        <switch-theme class="header-item theme"></switch-theme>
      </div>
    </div>
    <stream-preview v-if="!isMobile" ref="streamPreviewRef"></stream-preview>
    <room-control
      ref="roomControlRef"
      :given-room-id="givenRoomId"
      :user-name="userName"
      @update-user-name="handleUpdateUserName"
      @create-room="handleCreateRoom"
      @enter-room="handleEnterRoom"
    ></room-control>
  </div>
</template>

<script setup lang="ts">
import UserInfo from '@TUIRoom/components/RoomHeader/UserInfo/index.vue';
import StreamPreview from '@TUIRoom/components/RoomHome/StreamPreview.vue';
import RoomControl from '@TUIRoom/components/RoomHome/RoomControl/index.vue';
import LanguageIcon from '@TUIRoom/components/base/Language.vue';
import SwitchTheme from '@TUIRoom/components/base/SwitchTheme.vue';
import { checkNumber } from '@TUIRoom/utils/common';
import router from '@/router';
import { useRoute } from '@/router/wxRouter';
import { onMounted, Ref, ref } from 'vue';
import { getBasicInfo } from '../config/basic-info-config';
import { useI18n } from 'vue-i18n';
import { useBasicStore } from '@TUIRoom/stores/basic';
import { TUIRoomEngine } from '@tencentcloud/tuiroom-engine-wx';
import useGetRoomEngine from '@TUIRoom/hooks/useRoomEngine';
import { isMobile } from '@TUIRoom/utils/useMediaValue';
import logger from '@TUIRoom/utils/common/logger';

const route = useRoute();
const streamPreviewRef = ref();
const userName: Ref<string> = ref('');
const avatarUrl: Ref<string> = ref('');
const userId: Ref<string> = ref('');
const { t } = useI18n();
const roomEngine = useGetRoomEngine();
const basicStore = useBasicStore();
const roomControlRef = ref();

const roomId = checkNumber((route.query?.roomId) as string) ? route.query?.roomId : '';
const givenRoomId: Ref<string> = ref((roomId) as string);

function setTUIRoomData(action: string, mode?: string) {
  const roomParam = isMobile ? roomControlRef.value.getRoomParam() : streamPreviewRef.value.getRoomParam();
  const roomData = {
    action,
    roomMode: mode || 'FreeToSpeak',
    roomParam,
  };
  uni.setStorageSync('tuiRoom-roomInfo', JSON.stringify(roomData));
}

async function checkRoomExistWhenCreateRoom(roomId: string) {
  let isRoomExist = false;
  const tim = roomEngine.instance?.getTIM();
  try {
    await tim.searchGroupByID(roomId);
    isRoomExist = true;
  } catch (error: any) {
    // 房间不存在
  }
  return isRoomExist;
}

/**
 * Generate room number when creating a room
 *
 * 创建房间时生成房间号
**/
async function generateRoomId(): Promise<string> {
  const roomId = String(Math.ceil(Math.random() * 1000000));
  const isRoomExist = await checkRoomExistWhenCreateRoom(String(roomId));
  if (isRoomExist) {
    return await generateRoomId();
  }
  return roomId;
}
function handleUpdateUserName(userName: string) {
  try {
    const currentUserInfo = JSON.parse(uni.getStorageSync('tuiRoom-userInfo') as string);
    currentUserInfo.userName = userName;
    uni.setStorageSync('tuiRoom-userInfo', JSON.stringify(currentUserInfo));
  } catch (error) {
    logger.log('sessionStorage error', error);
  }
}
/**
 * Processing Click [Create Room]
 *
 * 处理点击【创建房间】
**/
async function handleCreateRoom(mode: string) {
  setTUIRoomData('createRoom', mode);
  const roomId = await generateRoomId();
  router.replace({
    path: 'room',
    query: {
      roomId,
    },
  });
}

/**
 * Processing Click [Enter Room]
 *
 * 处理点击【进入房间】
**/
async function handleEnterRoom(roomId: string) {
  setTUIRoomData('enterRoom');
  router.replace({
    path: 'room',
    query: {
      roomId,
    },
  });
}

/**
 * Processing users click [Logout Login] in the upper left corner of the page
 *
 * 处理用户点击页面左上角【退出登录】
**/
async function handleLogOut() {
/**
 * The accessor handles the logout method
 *
 * 接入方处理 logout 方法
**/
}

async function handleInit() {
  uni.removeStorageSync('tuiRoom-roomInfo');
  uni.removeStorageSync('tuiRoom-userInfo');
  let currentUserInfo = null;
  if (uni.getStorageSync('tuiRoom-userInfo')) {
    currentUserInfo = JSON.parse(uni.getStorageSync('tuiRoom-userInfo') as string);
  } else {
    currentUserInfo = await getBasicInfo();
    currentUserInfo && uni.setStorageSync('tuiRoom-userInfo', JSON.stringify(currentUserInfo));
  }
  basicStore.setBasicInfo(currentUserInfo);
  userName.value = currentUserInfo?.userName;
  avatarUrl.value = currentUserInfo?.avatarUrl;
  userId.value = currentUserInfo?.userId;
  const { sdkAppId, userSig } = currentUserInfo;
  /**
   * TUIRoomCore.checkRoomExistence method can only be used after logging into TUIRoomCore.
   *
   * 登录 TUIRoomCore, 只有登录 TUIRoomCore 之后，才可以使用 TUIRoomCore.checkRoomExistence 方法
  **/
  await TUIRoomEngine.login({ sdkAppId, userId: userId.value, userSig });
}

handleInit();

onMounted(() => {
  TUIRoomEngine.once('ready', () => {
    if (!isMobile) {
      streamPreviewRef.value.startStreamPreview();
    }
  });
});

</script>

<style>
@import '@TUIRoom/assets/style/white-theme.scss';
/* * {
    transition: background-color .5s,color .5s !important;
  } */
</style>

<style lang="scss" scoped>
.home-container-H5{
  width: 100%;
  height: 100%;
  // background: var(--background-color-style);
  color: #B3B8C8;
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: PingFangSC-Medium;
    .header-H5{
      width: 100%;
      position: absolute;
      top: 0;
      padding: 22px 24px;
      display: flex;
      align-items: center;
      justify-content: space-between;
  }
}
.home-container {
  width: 100%;
  height: 100%;
  background: var(--background-color-style);
  color: #B3B8C8;
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: PingFangSC-Medium;
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
  .container-icon{
    display: flex;
    align-items: center;
  }
</style>
