<template>
  <div v-if="!isMe" class="member-control-container">
    <div
      class="mute-btn"
      @click="singleControl.func(userInfo)"
    >
      {{ singleControl.title }}
    </div>
    <div class="more-container" @click="handlePopUp">
      <div class="more-btn" @click="toggleClickMoreBtn">
        {{ t('More') }}
        <svg-icon class="more-icon" :icon-name="ICON_NAME.ArrowBorderDown"></svg-icon>
      </div>
      <div
        v-show="showMoreControl"
        id="operate-list"
        :class="dropdownClass"
        @mouseleave="handleMouseLeave"
      >
        <div
          v-for="item, index in controlList"
          :key="index"
          class="user-operate-item"
          @click="item.func(userInfo)"
        >
          {{ item.title }}
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue';
import { useI18n } from '../../../locales';

import { UserInfo, useRoomStore } from '../../../stores/room';
import { ICON_NAME } from '../../../constants/icon';
import useGetRoomEngine from '../../../hooks/useRoomEngine';
import { useBasicStore } from '../../../stores/basic';
import SvgIcon from '../../common/SvgIcon.vue';
import useMasterApplyControl from '../../../hooks/useMasterApplyControl';
import { TUIMediaDevice } from '@tencentcloud/tuiroom-engine-electron';
const roomEngine = useGetRoomEngine();
const { t } = useI18n();

const basicStore = useBasicStore();
const roomStore = useRoomStore();
const dropdownClass = ref('');
/**
 * Functions related to the Raise Your Hand function
 *
 * 举手发言功能相关函数
**/
const {
  agreeUserOnStage,
  denyUserOnStage,
  inviteUserOnStage,
  cancelInviteUserOnStage,
  kickUserOffStage,
} = useMasterApplyControl();

interface Props {
  userInfo: UserInfo,
}

const props = defineProps<Props>();

const showMoreControl = ref(false);
const isMe = computed(() => basicStore.userId === props.userInfo.userId);
const isAnchor = computed(() => props.userInfo.onSeat === true);
const isAudience = computed(() => props.userInfo.onSeat !== true);
const isFreeSpeechMode = computed(() => roomStore.isFreeSpeakMode);
const isSpeakAfterTakingSeat = computed(() => roomStore.isSpeakAfterTakingSeatMode);
/**
* Control the centralized matching of elements
 *
 * Free speech mode
 * 1. Current user is the anchor.
 * Banning/unbanning Banning/unbanning drawing
 * Banning chat
 * Kicked out of the room
 *
 * Raise hand to speak mode
 * 1. Current user is the host.
 * Banning/unbanning Banning/unbanning drawing
 * Invite to step down
 * Ban chat
 * Kicked out of the room
 * 2.
 * 2. Audience did not request to be on the mic.
 * Invite on stage / Disinvite on stage * Ban chat
 * Kicked out of the room
 * 3.
 * 3. the audience is applying for the mic.
 * Agree to go on stage Deny go on stage
 * Ban chat
 * Kicked out of the room
 *
 * 控制元素的集中匹配情况
 *
 * 自由发言模式
 * 1.当前用户为主播：
 * 禁言/解除禁言/取消解除禁言  禁画/解除禁画/取消解除禁画
 *                         禁止聊天
 *                         踢出房间
 *
 * 举手发言模式
 * 1. 当前用户为主播：
 * 禁言/解除禁言/取消解除禁言  禁画/解除禁画/取消解除禁画
 *                         邀请下台
 *                         禁止聊天
 *                         踢出房间
 *
 * 2. 观众没有申请上麦：
 * 邀请上台/取消邀请上台   禁止聊天
 *                      踢出房间
 *
 * 3. 观众正在申请上麦：
 * 同意上台      拒绝上台
 *              禁止聊天
 *              踢出房间
**/
function handlePopUp() {
  const dropdown = document.getElementById('operate-list');
  const roomContainer = document.getElementById('roomContainer');
  const dropdownRectBottom = dropdown !== null && dropdown.getBoundingClientRect().bottom;
  const roomContainerHeight = roomContainer !== null && roomContainer.offsetHeight;
  if (dropdownRectBottom > roomContainerHeight) {
    dropdownClass.value = 'user-operate-list dropdownUp';
  } else {
    dropdownClass.value = 'user-operate-list';
  }
};

const singleControl = computed(() => {
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  const control = { title: '', func: (userInfo: UserInfo) => {} };
  if (isFreeSpeechMode.value) {
    control.title = props.userInfo.hasAudioStream ? t('Mute') : (props.userInfo.isRequestingUserOpenMic ? t('Cancel Unmute') : t('Unmute'));
    control.func = muteUserAudio;
  } else if (isSpeakAfterTakingSeat.value) {
    if (isAnchor.value) {
      control.title = props.userInfo.hasAudioStream ? t('Mute') : (props.userInfo.isRequestingUserOpenMic ? t('Cancel Unmute') : t('Unmute'));
      control.func = muteUserAudio;
    } else if (isAudience.value) {
      if (props.userInfo.isUserApplyingToAnchor) {
        control.title = t('Agree to the stage');
        control.func = agreeUserOnStage;
      } else {
        control.title = props.userInfo.isInvitingUserToAnchor ?  t('Cancel stage') : t('Invite stage');
        control.func = toggleInviteUserOnStage;
      }
    }
  }
  return control;
});

const denyOnStage = computed(() => ({ title: t('Refuse stage'), func: denyUserOnStage }));
const makeOffStage = computed(() => ({ title: t('Step down'), func: kickUserOffStage }));

const videoControlTitle = computed(() => {
  if (props.userInfo.hasVideoStream) {
    return t('Disable video');
  }
  if (props.userInfo.isRequestingUserOpenCamera) {
    return t('Cancel Enable video');
  }
  return t('Enable video');
});
const videoControl = computed(() => ({ title: videoControlTitle.value, func: muteUserVideo }));

const chatControlTitle = computed(() => (props.userInfo.isChatMutedByMaster ? t('Enable chat') : t('Disable chat')));
const forbidChat = computed(() => ({ title: chatControlTitle.value, func: disableUserChat }));

const kickUser = computed(() => ({ title: t('Kick out'), func: kickOffUser }));
const controlList = computed(() => {
  const list = [forbidChat.value, kickUser.value];
  if (isFreeSpeechMode.value) {
    list.unshift(videoControl.value);
  }
  if (isAnchor.value &&  isSpeakAfterTakingSeat.value) {
    list.unshift(videoControl.value);
    list.splice(1, 0, makeOffStage.value);
  }
  if (isAudience.value && props.userInfo.isUserApplyingToAnchor) {
    list.splice(0, 0, denyOnStage.value);
  }
  return list;
});

function toggleClickMoreBtn() {
  showMoreControl.value = !showMoreControl.value;
}
function handleMouseLeave() {
  showMoreControl.value = false;
}

/**
 * Invitation to the stage/uninvitation to the stage
 *
 * 邀请上台/取消邀请上台
**/
async function toggleInviteUserOnStage(userInfo: UserInfo) {
  const { isInvitingUserToAnchor } = userInfo;
  if (isInvitingUserToAnchor) {
    cancelInviteUserOnStage(userInfo);
  } else {
    inviteUserOnStage(userInfo);
  }
}

/**
 * Banning/Unbanning
 *
 * 禁麦/邀请打开麦克风
**/
async function muteUserAudio(userInfo: UserInfo) {
  if (userInfo.hasAudioStream) {
    await roomEngine.instance?.closeRemoteDeviceByAdmin({
      userId: userInfo.userId,
      device: TUIMediaDevice.kMicrophone,
    });
  } else {
    if (userInfo.isRequestingUserOpenMic) {
      const requestId = userInfo.requestUserOpenMicRequestId;
      requestId && await roomEngine.instance?.cancelRequest({ requestId });
      roomStore.setRequestUserOpenMic({ userId: userInfo.userId, isRequesting: false });
    } else {
      const request = await roomEngine.instance?.openRemoteDeviceByAdmin({
        userId: userInfo.userId,
        device: TUIMediaDevice.kMicrophone,
        timeout: 0,
        requestCallback: () => {
          // 处理请求超时，应答，拒绝的情况
          roomStore.setRequestUserOpenMic({ userId: userInfo.userId, isRequesting: false });
        },
      });
      if (request && request.requestId) {
        roomStore.setRequestUserOpenMic({ userId: userInfo.userId, isRequesting: true, requestId: request.requestId });
      }
    }
  }
}

/**
 * Banned painting/unbanned painting
 *
 * 禁画/取消禁画
**/
async function muteUserVideo(userInfo: UserInfo) {
  if (userInfo.hasVideoStream) {
    await roomEngine.instance?.closeRemoteDeviceByAdmin({
      userId: userInfo.userId,
      device: TUIMediaDevice.kCamera,
    });
  } else {
    if (userInfo.isRequestingUserOpenCamera) {
      const requestId = userInfo.requestUserOpenCameraRequestId;
      requestId && await roomEngine.instance?.cancelRequest({ requestId });
      roomStore.setRequestUserOpenCamera({ userId: userInfo.userId, isRequesting: false });
    } else {
      const request = await roomEngine.instance?.openRemoteDeviceByAdmin({
        userId: userInfo.userId,
        device: TUIMediaDevice.kCamera,
        timeout: 0,
        requestCallback: () => {
          // 处理请求超时，应答，拒绝的情况
          roomStore.setRequestUserOpenCamera({ userId: userInfo.userId, isRequesting: false });
        },
      });
      if (request && request.requestId) {
        roomStore.setRequestUserOpenCamera({
          userId: userInfo.userId,
          isRequesting: true,
          requestId: request.requestId,
        });
      }
    }
  }
}
/**
 * Allow text chat / Cancel text chat
 *
 * 允许文字聊天/取消文字聊天
**/
function disableUserChat(userInfo: UserInfo) {
  const currentState = userInfo.isChatMutedByMaster;
  roomStore.setMuteUserChat(userInfo.userId, !currentState);
  roomEngine.instance?.disableSendingMessageByAdmin({
    userId: userInfo.userId,
    isDisable: !currentState,
  });
}

/**
 * Kick the user out of the room
 *
 * 将用户踢出房间
**/
function kickOffUser(userInfo: UserInfo) {
  roomEngine.instance?.kickRemoteUserOutOfRoom({
    userId: userInfo.userId,
  });
}

</script>

<style lang="scss" scoped>
.member-control-container {
  display: flex;
  flex-direction: row;
  .more-btn, .mute-btn {
    height: 32px;
    border-radius: 2px;
    font-size: 14px;
    color: #FFFFFF;
    line-height: 32px;
    padding: 0 20px;
  }
  .mute-btn {
    background-image: linear-gradient(235deg, #1883FF 0%, #0062F5 100%);
  }
  .more-container {
    position: relative;
    .more-btn {
      display: flex;
      flex-direction: row;
      align-items: center;
      background: var(--more-btn-background);
      border: 1px solid #ADB6CC;
      margin-left: 12px;
      .more-icon {
        margin-left: 4px;
        width: 20px;
        height: 20px;
      }
    }
    .user-operate-list {
      position: absolute;
      padding: 4px 0;
      top: calc(100% + 10px);
      right: 0;
      min-width: 140px;
      background: var(--user-operate-list);
      border-radius: 4px;
      box-shadow: 0 1px 10px 0 rgba(0,0,0,0.30);
      .user-operate-item {
        cursor: pointer;
        padding: 0 20px;
        text-align: center;
        font-size: 14px;
        color: var(--user-operate-item);
        height: 40px;
        line-height: 40px;
        white-space: nowrap;
      }
    }
    .dropdownUp {
      transform: translateY(-140%);
    }
  }
}
</style>
