<template>
  <div>
    <div class="end-control-container">
      <div @tap="stopMeeting" class="end-button" tabindex="1">{{ t('EndH5') }}</div>
    </div>
    <div v-if="visible" class="end-main-content">
      <div :class="isShowLeaveRoomDialog ? 'end-dialog-leave':'end-dialog-dismiss'">
        <div v-if="currentDialogType === DialogType.BasicDialog">
          <div v-if="roomStore.isMaster" class="end-dialog-header">
            <span v-if="roomStore.isMaster" class="end-dialog-text">
              <!-- eslint-disable-next-line max-len -->
              {{ t('If you do not want to end the meeting, please designate a new host before leaving the meeting.') }}
            </span>
            <span v-else>{{ t('Are you sure you want to leave this room?') }}</span>
          </div>
        </div>
        <div v-if="currentDialogType === DialogType.BasicDialog" class="dialog-middle-content">
          <span
            v-if="roomStore.isMaster"
            :class="isShowLeaveRoomDialog ? 'end-button-dismiss':'end-button-dismiss-single'"
            @click.stop="dismissRoom"
          >
            {{ t('Dismiss') }}
          </span>
          <span
            v-if="isShowLeaveRoomDialog"
            @tap="leaveRoom"
            :class="roomStore.isMaster ?'end-button-leave':'end-button-leave-single'"
          >
            {{ t('Leave') }}
          </span>
          <span class="end-button-cancel" @click.stop="cancel">{{ t('Cancel') }}</span>
        </div>
        <div v-if="currentDialogType === DialogType.TransferDialog">
          <span class="end-button-cancel" @click.stop="cancel">{{ t('Cancel') }}</span>
        </div>
      </div>
    </div>
    <popup
      v-show="showSideBar"
      :title="t('Appoint a new host')"
      class="transfer-container"
    >
      <template #sidebarContent>
        <div style="height:100%">
          <div>
            <div class="transfer-header">
              <input
                v-model="searchName"
                type="text"
                class="searching-input"
                :placeholder="t('Search for conference attendees')"
                enterkeyhint="done"
              >
            </div>
            <div class="transfer-body">
              <div
                v-for="user in filteredList"
                :key="user.userId"
                class="transfer-list-content"
                @click="handleShowMemberControl(user.userId)"
              >
                <div class="member-basic-info">
                  <Avatar class="avatar-url" :img-src="user.avatarUrl"></Avatar>
                  <div class="user-name">{{ user.userName || user.userId }}</div>
                  <svg-icon
                    v-if="selectedUser === user.userId"
                    class="correct"
                    icon-name="correct"
                    size="custom"
                  ></svg-icon>
                </div>
              </div>
              <div v-if="hasNoData" class="member-hasNoData">{{ t('No relevant user found.') }}</div>
            </div>
          </div>
        </div>
      </template>
      <template #sidebarFooter>
        <div class="transfer-leave" @click="transferAndLeave">
          <span class="transfer-button">{{ t('Transfer and leave') }}</span>
        </div>
      </template>
    </popup>
  </div>
</template>

<script setup lang="ts">
import { onUnmounted } from 'vue';
import { ElMessageBox, ElMessage } from '../../../elementComp';
import { TUIRoomEngine, TUIRole, TUIRoomEvents } from '@tencentcloud/tuiroom-engine-wx';
import useEndControl from './useEndControlHooks';
import logger from '../../../utils/common/logger';
import popup from '../../common/PopUpH5.vue';
import SvgIcon from '../../common/SvgIcon.vue';
import Avatar from '../../base/Avatar.vue';

const {
  t,
  isShowLeaveRoomDialog,
  roomStore,
  basicStore,
  roomEngine,
  localUser,
  stopMeeting,
  cancel,
  DialogType,
  logPrefix,
  currentDialogType,
  visible,
  closeMediaBeforeLeave,
  resetState,
  toggleMangeMemberSidebar,
  searchName,
  hasNoData,
  handleShowMemberControl,
  filteredList,
  selectedUser,
  showSideBar,
} = useEndControl();


const emit = defineEmits(['on-exit-room', 'on-destroy-room']);


/**
 * Active room dismissal
 *
 * 主动解散房间
**/
async function dismissRoom() {
  try {
    logger.log(`${logPrefix}dismissRoom: enter`);
    await closeMediaBeforeLeave();
    await roomEngine.instance?.destroyRoom();
    resetState();
    emit('on-destroy-room', { code: 0, message: '' });
  } catch (error) {
    logger.error(`${logPrefix}dismissRoom error:`, error);
  }
}

/**
 * Leave the room voluntarily
 *
 * 主动离开房间
**/
async function leaveRoom() { // eslint-disable-line
  try {
    if (roomStore.isMaster) {
      toggleMangeMemberSidebar();
      resetState();
      return;
    }
    await closeMediaBeforeLeave();
    const response = await roomEngine.instance?.exitRoom();
    logger.log(`${logPrefix}leaveRoom:`, response);
    resetState();
    emit('on-exit-room', { code: 0, message: '' });
  } catch (error) {
    logger.error(`${logPrefix}leaveRoom error:`, error);
  }
}

async function transferAndLeave() {
  if (!selectedUser.value) {
    return;
  }
  try {
    const userId = selectedUser.value;
    const changeUserRoleResponse = await roomEngine.instance?.changeUserRole({ userId, userRole: TUIRole.kRoomOwner });
    logger.log(`${logPrefix}transferAndLeave:`, changeUserRoleResponse);
    await closeMediaBeforeLeave();
    const exitRoomResponse = await roomEngine.instance?.exitRoom();
    logger.log(`${logPrefix}exitRoom:`, exitRoomResponse);
    basicStore.setSidebarOpenStatus(false);
    basicStore.setSidebarName('');
    resetState();
    emit('on-exit-room', { code: 0, message: '' });
  } catch (error) {
    logger.error(`${logPrefix}transferAndLeave error:`, error);
  }
}

/**
 * notification of room dismissal from the host
 *
 * 收到主持人解散房间通知
**/
const onRoomDismissed = async (eventInfo: { roomId: string}) => {
  try {
    const { roomId } = eventInfo;
    logger.log(`${logPrefix}onRoomDismissed:`, roomId);
    ElMessageBox.alert(t('The host closed the room.'), t('Note'), {
      customClass: 'custom-element-class',
      confirmButtonText: t('Confirm'),
      appendTo: '#roomContainer',
      callback: async () => {
        resetState();
        emit('on-destroy-room', { code: 0, message: '' });
      },
    });
  } catch (error) {
    logger.error(`${logPrefix}onRoomDestroyed error:`, error);
  }
};

/**
 * By listening for a change in ownerId,
 * the audience receives a notification that the host has handed over the privileges
 *
**/

const onUserRoleChanged = async (eventInfo: {userId: string, userRole: TUIRole }) => {
  if (eventInfo.userRole === TUIRole.kRoomOwner) {
    const { userId } = eventInfo;
    let newName = roomStore.getUserName(userId) || userId;
    if (userId === localUser.value.userId) {
      newName = t('me');
    }
    const tipMessage = `${t('Moderator changed to ')}${newName}`;
    ElMessage({
      type: 'success',
      message: tipMessage,
    });
    if (roomStore.localUser.userId === userId) {
      roomStore.setLocalUser({ userRole: TUIRole.kRoomOwner });
    } else {
      roomStore.setRemoteUserRole(userId, TUIRole.kRoomOwner);
    }
    roomStore.setMasterUserId(userId);
    resetState();
    if (roomStore.isAnchor) return;
    if (roomStore.isSpeakAfterTakingSeatMode) {
      await roomEngine.instance?.takeSeat({ seatIndex: -1, timeout: 0 });
    };
  }
};

TUIRoomEngine.once('ready', () => {
  roomEngine.instance?.on(TUIRoomEvents.onRoomDismissed, onRoomDismissed);
  roomEngine.instance?.on(TUIRoomEvents.onUserRoleChanged, onUserRoleChanged);
});

onUnmounted(() => {
  roomEngine.instance?.off(TUIRoomEvents.onRoomDismissed, onRoomDismissed);
  roomEngine.instance?.off(TUIRoomEvents.onUserRoleChanged, onUserRoleChanged);
});

</script>
<style lang="scss" scoped>
@import '../../../assets/style/var.scss';
.end-control-container{
  .end-button {
    border: 1px solid #FF2E2E;
    border-radius: 4px;
    font-weight: 400;
    font-size: 12px;
    color: #FF2E2E;
    letter-spacing: 0;
    cursor: pointer;
    text-align: center;
    line-height: 21px;
    padding:  0 8px;
    &:hover {
      background-color: #FF2E2E;
      color: $whiteColor;
    }
  }
}
.dialog-middle-content{
  width: 100%;
  display: flex;
  flex-direction: column;
}
.end-main-content{
  position: fixed;
  left: 0;
  top: 0;
  bottom: 0;
  width: 100vw;
  height: auto;
  box-sizing: border-box;
  background-color: var(--log-out-mobile);
  backdrop-filter: blur(10px);
  .end-dialog-leave,.end-dialog-dismiss {
    width: 90%;
    border-radius: 14px;
    position: fixed;
    right: 0;
    left: 0;
    bottom: 10vh;
    z-index: 9;
    margin: auto;
    .manage-transfer{
      position: absolute;
      top: 0;
    }
    .end-dialog-header {
      background: #CACACB;
      font-family: 'PingFang SC';
      font-style: normal;
      font-weight: 500;
      font-size: 14px;
      line-height: 17px;
      text-align: center;
      color: #6a6c74;
      height: 56px;
      width: 100%;
      margin: 0 auto;
      border-top-left-radius: 14px;
      border-top-right-radius: 14px;
      border-bottom: .5px solid #4f4e4e;
      display: flex;
      justify-content: center;
      align-items: center;
      .end-dialog-text {
        width: 230px;
      }
    }
    .end-button-dismiss,.end-button-leave,.end-button-cancel,.end-button-dismiss-single,.end-button-leave-single {
      width: 100%;
      font-family: 'PingFang SC';
      font-style: normal;
      font-weight: 400;
      font-size: 20px;
      line-height: 24px;
      text-align: center;
      color: #FF2E2E;
      border-style: none;
      background: #CACACB;
      padding: 20px 0;
    }
    .end-button-leave {
      color: #007AFF;
      border-bottom-left-radius: 14px;
      border-bottom-right-radius: 14px;
      border-top: .5px solid #8f8e8e;
    }
    .end-button-leave-single{
      position: absolute;
      bottom: 1vh;
      border-radius: 14px;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .end-button-cancel {
      position: absolute;
      bottom: -9vh;
      left: 0;
      background-color: #FFFFFF;
      border-radius: 14px;
      color: #007AFF;
      font-weight: 600;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .end-button-dismiss-single {
      border-bottom-left-radius: 14px;
      border-bottom-right-radius: 14px;
      display: flex;
      align-items: center;
      justify-content: center;
    }
  }
  .end-dialog-dismiss {
    bottom: 12vh;
    .end-button-cancel{
      bottom: -9vh;
    }
  }
}
.transfer-container {
  position: fixed;
  left: 0;
  top: 0;
  height: 100vh;
  z-index: 102;
  .transfer-header {
    display: flex;
    justify-content: center;
     input[type=text]{
        background-image: url(../../../assets/icons/svg/search.svg);
        background-repeat: no-repeat;
        background-position: 10px center;
        padding-left: 35px;
        background-color: var(--transfer-input-color-h5);
     }
        .searching-input {
          padding: 0 80px;
          background: #292D38;
          border-radius: 8px;
          height: 34px;
          border-style: none;
          font-family: 'PingFang SC';
          font-style: normal;
          font-weight: 450;
          font-size: 16px;
          line-height: 18px;
          color: #676c80;
          caret-color: var(--caret-color);
        ::placeholder {
          font-family: 'PingFang SC';
          font-style: normal;
          font-weight: 400;
          font-size: 16px;
          line-height: 18px;
          color: #676c80;
        }
        &:focus-visible {
         outline: none;
       }
      }
  }
  .transfer-body {
    flex: 1;
    overflow-y: scroll;
    .transfer-list-content{
      padding-bottom: 5px;
    .transfer-item-container {
      display: flex;
      flex-direction: row;
      align-items: center;
      height: 69px;
      justify-content: space-between;
      padding: 0 32px;
    }
    .member-basic-info {
      display: flex;
      position: relative;
      width: 100%;
      flex-direction: row;
      align-items: center;
      padding: 24px 20px 0 24px;
        .avatar-url {
          width: 40px !important;
          height: 40px !important;
          border-radius: 50%;
        }
        .user-name {
          margin-left: 9px;
          color: var(--input-font-color);
          max-width: 70% !important;
          white-space: nowrap;
          text-overflow: ellipsis;
          overflow: hidden;
          font-family: 'PingFang SC';
          font-style: normal;
          font-weight: 500;
          font-size: 16px !important;
          line-height: 22px;
          letter-spacing: -0.24px;
          display: flex;
        }
        .correct {
          width: 24px;
          height: 16px;
          position: absolute;
          right: 5vw;
          background-size: cover;
        }
      }
    }
    .member-hasNoData{
      position: fixed;
      top: 30%;
      left: 30%;
      width: 40vw;
      height: 5vh;
      color: var(--input-font-color);
      border-radius: 4px;
      background-color: var(--message-list-color);
      display: flex;
      justify-content: center;
      align-items: center;
    }
  }
}
.transfer-leave{
  position: fixed;
  bottom: 2vh;
  left: 7vw;
  display: flex;
  align-items: center;
  justify-content: center;
  .transfer-button{
    width: 86vw;
    height: 5vh;
    background: linear-gradient(315deg, #006EFF 0%, #0C59F2 98.81%);
    border-radius: 8px;
    font-family: 'PingFang SC';
    font-style: normal;
    font-weight: 500;
    font-size: 16px;
    line-height: 22px;
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
    color: #FFFFFF;
    border-style: none;
  }
}
</style>
