<template>
  <div class="relative" :class="[
    hasAction ? 'g-only-event-msg bottom-line' : 'g-content-card-v2 bottom-line',
    { active: '#' + tid === $route.hash && !isReviewComment },
    { 'jump-comment-focus': !isReviewComment && jumpCommentId === props.targetId },
    $attrs?.class
  ]" :id="!isReviewComment ? tid : ''" :key="discussion_id">
    <div :class="hasAction ? 'g-discussion-header g-discussion-header-only-event-msg' : 'g-discussion-header'">
      <div class="flex items-center gap-[8px] overflow-hidden w-full">
        <!-- 事件消息 - icon -->
        <div v-if="eventIconName" class="g-discussion-status">
          <Icon :name="eventIconName" />
        </div>
        <div class="g-discussion-header-info">
          <GLink :to="{ name: 'homepage', params: { namespace: cAuthor?.username } }" :disabled="!cAuthor?.username"
            class="avatar-link">
            <GAvatar :src="cAuthor?.avatar_url || cAuthor?.avatar || cAuthor?.avatarUrl" :width="24" :height="24"
              :name="pickNickName(cAuthor)" />
          </GLink>
          <GLink :to="{ name: 'homepage', params: { namespace: cAuthor?.username } }" :disabled="!cAuthor?.username"
            class="user-link" :title="pickNickName(cAuthor)">
            {{ discussionAuthorName }}
          </GLink>
          <span v-if="isMember" class="g-discussion-permission">{{ role || $t('repoSetting.member.titleName') }}</span>
          <!-- 代码检视意见，特有 header info 部分-->
          <span v-if="isReviewComment" class="file-change-info-group">
            <GLink :to="{
              name: 'repoMergeDetailDiffs',
              query: {
                file: noteJumpInfo.diffFile,
                version: isCodeReviewComment ? noteJumpInfo.patchSetIid : (noteJumpInfo.patchSetIid || 0),
                expired: isCodeReviewComment ? noteJumpInfo.isExpired : (noteJumpInfo.isExpired || false)
              }
            }" class="diff-file-name" :title="diffFile" @click.stop="handleDiffFileClick">
              {{ diffFile }}
            </GLink>
            <d-tooltip :content="copyTip">
              <Icon name="gt-copy" @click="onCopyPath" class="btn" />
            </d-tooltip>
            <span v-if="isCodeReviewComment && (addedLines !== undefined)" class="text-[var(--theme-success)]">+{{
              addedLines }}</span>
            <span v-if="isCodeReviewComment && (removedLines !== undefined)" class="text-[var(--theme-danger)]">-{{
              removedLines || 0}}</span>
            <span v-if="isCodeReviewComment" class="code-review-switch" @click="isFold = !isFold">
              <d-icon :name="isFold ? 'icon-chevron-down' : 'icon-chevron-up'" class="cursor-pointer"></d-icon>
            </span>
          </span>
          <div class="hidden md:flex items-center overflow-hidden flex-1">
            <span class="create-time leading-[20px]" v-if="onlyShowTime">
              <Time :time="createdAt"></Time>
            </span>
            <!-- 评论修改标题这些会有超长换行的，测试提出需要和时间成一个整体 -->
            <EventMsg v-if="onlyShowEventMsg" class="event-msg-wrap" :type="type" :action="eventType"
              :dateTime="createdAt" :time="`${formatTimeWithMonthAndYear(new Date(createdAt || ''))}`" :body="eventMsg"
              :key="data?.id" :data="data" :labelList="labelList" :enterpriseLabelList="enterpriseLabelList" />
          </div>
        </div>
        <!-- 时间 编辑按钮 更多菜单 -->
        <div class="g-discussion-header-right">
          <!-- 评论/回复，历史修改记录查看 -->
          <d-dropdown v-if="showVersionDropdown" :visible="versionDropdownVisible"
            :position="isPhone() ? ['bottom', 'top-start'] : ['bottom-end', 'top-start']" :align="'start'"
            close-scope="blank" @toggle="(blo: boolean) => { versionToggleStatus = blo }"
            overlay-class="content-version-dropdown__wrapper">
            <d-button variant="text" class="mr-2" @click.stop="versionDropdownVisible = true">
              <div class="content-version-dropdown">
                {{ $t('repo.issueVersion.trigger') }}
                <Icon name="gt-line-down" size="14px" color="var(--theme-icon-fill)" class="ml-[4px]" :class="{
                  'arrow-down': !versionToggleStatus,
                  'arrow-up': versionToggleStatus
                }" />
              </div>
            </d-button>
            <template #menu>
              <ContentVersionList :resourceId="praiseResourceId" :projectId="repoId" :type="discussionType"
                @view-diff="handleVersionDiffView" />
            </template>
          </d-dropdown>
          <!-- 文件变更，特有 header info 部分 -->
          <template v-if="isReviewComment">
            <!-- 评论数 -->
            <d-tooltip content="评论数量">
              <span class="whitespace-nowrap font-color-t2 flex items-center gap-[4px] mr-[8px]">
                <Icon name="gt-line-discussions" style="color: var(--devui-icon-fill-weak)" />&nbsp;
                <Number :number="noteNum || 0"></Number>
              </span>
            </d-tooltip>
            <!-- 转码 -->
            <!-- <d-tooltip content="转码"><d-button variant="text" :disabled="formatIng" @click="$emit('doFormat')" ><Icon name="gt-preview" /></d-button></d-tooltip> -->
            <!-- 跳转 -->
            <d-tooltip :content="$t('repo.commit.viewSourceCode')" v-if="sourceBranch && isCodeReviewComment">
              <d-button variant="text" @click="handleViewSource()">
                <Icon name="gt-line-code" style="color: var(--devui-icon-fill-weak)" />
              </d-button>
            </d-tooltip>

            <!-- 全屏 -->
            <d-tooltip v-if="isCodeReviewComment" :content="$t('pipeline.fullScreen')">
              <d-button variant="text" @click="handleExpand">
                <Icon :operable="true" name="gt-line-maximize" style="color: var(--devui-icon-fill-weak)" />
              </d-button>
            </d-tooltip>
          </template>
          <!-- 解决问题 -->
          <div v-if="showResolveSwitch" v-auth="{ authority: AuthMap.note.resolve }" class="resolved-group">
            <d-switch :disabled="!canResolve" @change="$emit('resolve-stat-change', isResolved)"
              :color="isResolved ? '#2951E0' : undefined" v-model="isResolved" size="sm" class="ques-switch">{{
                $t('repo.code.codeMerga.text1') }}</d-switch>
            <span>{{ isResolved ? $t('repo.code.codeMerga.text2') : $t('repo.code.codeMerga.text1') }}</span>
            <div class="w-1 h-4" style="border-right: 1px solid var(--theme-line-border)"></div>
          </div>
          <!-- 编辑 -->
          <template v-if="canEditDiscussion">
            <d-button v-if="editIng" variant="text" disabled>
              <Icon disabled :operable="true" name="gt-line-edit" color="var(--theme-disable-text)" />
            </d-button>
            <d-tooltip v-else :content="$t('pipeline.edit')">
              <d-button variant="text" @click="handleEdit">
                <Icon :operable="true" name="gt-line-edit" />
              </d-button>
            </d-tooltip>
          </template>

          <!-- 下拉操作菜单 -->
          <d-dropdown style="width: 104px" :align="'start'" v-if="body && !diff">
            <d-button variant="text">
              <d-tooltip :content="$t('pipeline.more')">
                <Icon :operable="true" name="gt-more-operate" style="color: var(--devui-icon-fill-weak)" />
              </d-tooltip>
            </d-button>
            <template #menu>
              <DOption class="text-center" @click="onCopy">{{ $t('repo.issue.copyLink') }}</DOption>
              <DOption class="text-center" @click="quoteReply" v-if="!isArchived && canQuoteReply"
                v-auth="{ authority: AuthMap.note.create }">{{ $t('repo.issue.quoteReply') }}</DOption>
              <DOption class="text-center" v-if="canDelete" v-intercept-click @click="onBeforeDelete"
                @mouseenter="renderModal = true">{{ $t('repo.issue.delete') }}</DOption>
            </template>
          </d-dropdown>
          <Icon v-if="isAcceptedSuggestion" @click="handleSuggestionExpand"
            :name="!suggestionExpand ? 'gt-line-Close-all' : 'gt-line-expand-all'" :operable="true">
            <template #suffix>
              <span>{{ suggestionExpand ? $t('repo.pr.fold') : $t('repo.pr.unfold') }}</span>
            </template>
          </Icon>
        </div>
      </div>
      <div class="flex md:hidden items-center">
        <span class="create-time leading-[20px]" v-if="onlyShowTime">
          <Time :time="createdAt"></Time>
        </span>
        <EventMsg v-if="onlyShowEventMsg" class="event-msg-wrap" :type="type" :action="eventType" :dateTime="createdAt"
          :time="`${formatTimeWithMonthAndYear(new Date(createdAt || ''))}`" :body="eventMsg" :key="data?.id"
          :data="data" :labelList="labelList" :enterpriseLabelList="enterpriseLabelList" />
      </div>
    </div>

    <div v-if="!isAcceptedSuggestion || suggestionExpand">
      <div class="g-discussion-divide-line" v-if="diffFile || diff || body"></div>

      <!-- 文件变动 -->
      <div :id="'code-review-' + diffFile" class="code-review-box diff-view" v-if="isCodeReviewComment"
        v-loading="formatIng"
        :class="{ 'diff-view-dark': currentTheme === 'black', 'diff-view': currentTheme === 'white' }">
        <d-code-review :fold="isFold" :diff="`--- a/a\n+++ b/${diffFile}\n` + diff" output-format="line-by-line"
          :allow-comment="false" :allow-expand="false" :options="diffOptions">
        </d-code-review>
      </div>

      <div class="outside" :class="[{ active: '#' + tid === $route.hash && isReviewComment }]">
        <!-- diff 评审 头部 -->
        <div class="g-discussion-header diff-header" v-if="diff" :id="isReviewComment ? tid : ''">
          <section class="comment-info">
            <div class="g-discussion-status"></div>
            <div class="g-discussion-header-info" :title="pickNickName(cAuthor)">
              <GLink :to="{ name: 'homepage', params: { namespace: cAuthor?.username } }" :disabled="!cAuthor?.username"
                class="avatar-link">
                <GAvatar :src="cAuthor?.avatar_url || cAuthor?.avatar || cAuthor?.avatarUrl" :width="20" :height="20"
                  :name="pickNickName(cAuthor)" />
              </GLink>
              <GLink :to="{ name: 'homepage', params: { namespace: cAuthor?.username } }" class="user-link"
                :disabled="!cAuthor?.username">
                {{ pickNickName(cAuthor) }}
              </GLink>
              <span class="create-time">
                <Time :time="createdAt + ''"></Time>
              </span>
              <span class="header-info" v-if="eventMsg">
                {{ eventMsg }}
              </span>
            </div>
            <!-- 时间 编辑按钮 更多菜单 -->
            <div class="g-discussion-header-right">
              <!-- 检视意见是否过期 -->
              <OutdateTag v-if="isExpiredSuggestion" class="mr-2" />
              <!-- 解决问题 -->
              <div v-if="resolvable && !data?.is_reply" class="resolved-group">
                <d-switch :disabled="!canResolve" @change="$emit('resolve-stat-change', isResolved)"
                  :color="isResolved ? '#2951E0' : undefined" v-model="isResolved" class="ques-switch" size="sm">{{
                    $t('repo.code.codeMerga.text1') }}</d-switch>
                <span>{{ isResolved ? $t('repo.code.codeMerga.text2') : $t('repo.code.codeMerga.text1') }}</span>
                <div class="w-1 h-4" style="border-right: 1px solid var(--theme-line-border)"></div>
              </div>
              <!-- 编辑 -->
              <d-tooltip :content="$t('pipeline.edit')" v-if="canEdit && !isArchived">
                <d-button variant="text" :disabled="editIng" @click="handleEdit">
                  <Icon :operable="true" name="gt-line-edit" style="color: var(--devui-icon-fill-weak)" />
                </d-button>
              </d-tooltip>

              <!-- 菜单 -->
              <d-dropdown style="width: 104px" :align="'start'" v-if="body">
                <d-button variant="text">
                  <Icon :operable="true" name="gt-more-operate" style="color: var(--devui-icon-fill-weak)" />
                </d-button>
                <template #menu>
                  <DOption class="text-center" @click="onCopy">{{ $t('repo.issue.copyLink') }}</DOption>
                  <DOption class="text-center" @click="quoteReply" v-if="!isArchived && canQuoteReply"
                    v-auth="{ authority: AuthMap.note.create }">{{ $t('repo.issue.quoteReply') }}</DOption>
                  <DOption class="text-center" v-if="canDeleteIssue" @click="onBeforeDelete"
                    @mouseenter="renderModal = true">{{ $t('repo.issue.delete') }}</DOption>
                </template>
              </d-dropdown>
            </div>
          </section>
          <section v-if="lineRangeVisible">
            <!-- 多行评论信息 -->
            <p class="pl-[24px] mt-[8px] text-[14px]">
              <span class="mr-[8px]">{{ $t('repo.pr.comment.range') }}</span>
              <span class="font-bold" :class="{
                'text-[var(--theme-success)]': lineRangeConfig.startLinePrefix === '+',
                'text-[var(--theme-danger)]': lineRangeConfig.startLinePrefix === '-'
              }">{{ `${lineRangeConfig.startLinePrefix}${lineRangeConfig.startLineNum}` }}</span>
              <span class="mx-[8px]">{{ t('activity.to') }}</span>
              <span class="font-bold" :class="{
                'text-[var(--theme-success)]': lineRangeConfig.endLinePrefix === '+',
                'text-[var(--theme-danger)]': lineRangeConfig.endLinePrefix === '-'
              }">{{ `${lineRangeConfig.endLinePrefix}${lineRangeConfig.endLineNum}` }}</span>
            </p>
          </section>
        </div>
        <!-- 描述 评论 编辑或展示-->
        <div class="g-discussion-body" :class="[
          body || editIng || $slots.noDescription ? 'p-[16px]' : 'p-[0px]',
          { 'jump-comment-focus': jumpCommentId === props.targetId || issueJumpCommentId === props.targetId }
        ]">
          <template v-if="!editIng">
            <MdRender :class="{ 'check-box-disabled': loading }" @accept-suggestion="handleAcceptSuggestion"
              :suggestion-parse="customParse" :id="'render_' + targetId" :content="mdBody"
              @md-render-change="handleMdRenderChange" @md-check-change="changeContentChecked" :canUseChecked="isAuthor"
              :isSuggestion="isSuggestion"></MdRender>
          </template>
          <template v-if="editIng">
            <MdEditor v-model="editText" :hint-config="hintConfig" ref="RefMd" :options="{ autofocus: true }"
              :project-id="decodeURIComponent(repoId as string)" toggle-repo-permission border-light
              :code-suggestion="isReviewComment" :custom-parse="customParse" :line-code="lineCodeInject"
              :uploading="uploading" :show-file-upload="showFileUpload && !!attachmentType" :file-upload="fileUpload"
              resize="vertical" :showCount="true" :maxLength="editMaxLength"></MdEditor>
            <slot name="noDescription" v-if="!body && !editIng"></slot>
          </template>

          <div class="edit-area relative z-10 justify-between" v-show="editIng">
            <div class="flex items-center gap-4">
              <d-button v-if="showFileUpload && !!attachmentType" @click="handleFileUpload" :loading="uploading"
                :disabled="uploading">
                <span class="mr-[8px]">{{ $t('repo.issue.attachment') }}</span>
                <Icon name="gt-line-uploadfiles" v-if="!uploading" color="var(--devui-text)" />
              </d-button>
            </div>
            <div class="edit-area_actions flex-row-reverse flex items-center gap-4">
              <d-button @click="$emit('save-content', editText)"
                :disabled="!editText.trim() || editText === realDiscussion || editText.length > editMaxLength"
                variant="solid" color="primary" :loading="loading">{{ $t('repoSetting.save') }}</d-button>
              <d-button @click="editIng = false">{{ $t('repo.code.cancelBtn') }}</d-button>
            </div>
          </div>
          <div v-show="!editIng && discussionType">
            <Praise :type="discussionType" :resource_id="praiseResourceId" :List="discussPraiseEmojiList"
              :autoComplete="props.autoComplete" :class="isPrCodeDiff && diffFile ? 'add-praise-bottom' : ''" />
          </div>
        </div>
      </div>

      <!-- 二级评论 -->
      <slot></slot>

      <!-- 回复的 -->
      <div class="discussion-reply-group" v-if="isReviewComment">
        <div v-show="replyIng">
          <MdEditor v-model="replyText" :placeholder="$t('repo.code.codeMerga.text3')" ref="RefReplyMD"
            :options="{ autofocus: true }" :project-id="decodeURIComponent(repoId as string)"
            :maxLength="replayMaxLength" :showCount="true" toggle-repo-permission border-light
            :hint-config="hintConfig"></MdEditor>
        </div>
        <div class="discussion-reply-action" v-show="replyIng">
          <d-button @click="handleReplyComment"
            :disabled="replyLoading || !replyText.trim() || replyText.length > replayMaxLength" variant="solid"
            color="primary" class="w-[80px]" :loading="replyLoading">{{ $t('repo.code.codeMerga.text3') }}</d-button>
          <d-button @click="replyIng = false" class="w-[80px]">{{ $t('repo.code.cancelBtn') }}</d-button>
        </div>
        <d-input v-if="!replyDisabled" :placeholder="$t('repo.code.codeMerga.text3')" @focus="replyIng = true"
          v-show="!replyIng"></d-input>
      </div>
    </div>

    <d-modal v-model="deleteModalVisible" v-if="renderModal">
      <template #header>
        <d-modal-header>
          <div class="flex items-center gap-2">
            <d-icon name="icon-warning-o" color="var(--color-warning,'#F5AB0B')"></d-icon>
            <span>{{ $t('common.tipText') }}</span>
          </div>
        </d-modal-header>
      </template>
      <p class="break-all whitespace-normal max-h-[10rem] overflow-auto">
        {{ $t('repo.pr.comment.deleteConfirm', { comment: originalComment }) }}
      </p>
      <template #footer>
        <div class="g-modal-footer">
          <d-button @click="deleteModalVisible = false">{{ $t('repo.code.cancelBtn') }}</d-button>
          <d-button @click="confirmDelete" autofocus :loading="delIng" color="danger">{{ $t('common.confirm')
            }}</d-button>
        </div>
      </template>
    </d-modal>

    <SuggestionCommitModal v-if="props.discussionType === COMMENT.MR" v-model:visible="commitModelVisible"
      :scene="props.discussionType" :content='commitContent' :noteId='targetId' :diffFile="props.diffFile"
      :mergeRequestIid="iid" :projectId="repoId" @success="onSuccessCommit" />
    <!-- 描述/评论，版本历史对比弹窗 -->
    <ContentDiffModal v-model="diffModalVisible" :currentItem="diffModalData.currentItem"
      :previousItem="diffModalData.previousItem" :projectId="repoId" :type="props.discussionType"
      :canVersionDelete="canVersionDelete" @delete="handleDeleteVersion"></ContentDiffModal>
    <!-- 描述/评论，版本历史删除弹窗 -->
    <GModal v-model="delVersionVisible" :showCloseIcon="false" :showFooter="false" :width="'340px'"
      :showWarnIconTriangle="!delVersionInfo.canDelete" :showDeleteIcon="delVersionInfo.canDelete"
      :title="delVersionInfo.title" class="version-del-modal">
      <p>{{ delVersionInfo.tip01 }}</p>
      <p>{{ delVersionInfo.tip02 }}</p>
      <div v-if="delVersionInfo.canDelete" class="mt-4 text-right">
        <d-button @click="delVersionVisible = false">{{ $t('repo.code.cancelBtn') }}</d-button>
        <d-button class="ml-4" @click="confirmVersionDelete" autofocus :loading="versionDelIng" color="danger">{{
          $t('common.delete') }}</d-button>
      </div>
      <div v-else class="mt-4 text-right">
        <d-button @click="delVersionVisible = false">{{ $t('common.confirm') }}</d-button>
      </div>
    </GModal>
  </div>
</template>
<script lang="ts" setup>
defineOptions({
  name: 'DiscussionItem',
  inheritAttrs: false
});
import { ref, watch, nextTick, computed, inject } from 'vue';
import vInterceptClick from '@/directives/intercept-click';
import { useRouter, useRoute } from 'vue-router';
import type { NoteJumpInfo } from '@/views/Repo/Merge/utils/types';
import type { IAuthor } from '@/api/issue/types';
import { RiskLevel } from '@/types/user.d';
import { getOriginalComment } from '@/api/repo';
import { deleteVersion } from '@/api/issue';
import { COMMENT } from '@/constant';
import GAvatar from '@/components/GAvatar/index.vue';
import Time from '@/components/Time/index.vue';
import MdEditor from '@/components/MdEditor/index.vue';
import EventMsg from '@/views/Repo/components/EventMsg';
import MdRender from '@/components/MdRender/index.vue';
import SuggestionCommitModal from '@/views/Repo/Merge/components/SuggestionCommitModal.vue';
import GLink from '@/components/GLink/index.vue';
import Number from '@/components/Number/index.vue';
import Icon from '@/components/Icon/index.vue';
import OutdateTag from './OutdateTag.vue';
import { useClipboard } from '@vueuse/core';
import isPhone from '@/utils/isPhone';
import { fullscreen, pickNickName, formatQuoteReply, formatAtAuthor } from '@/utils';
import { storeToRefs } from 'pinia';
import { repoInfoStore } from '@/stores/Repo';
import { useAccountStore } from '@/stores/user';
import { usePraiseDataStore } from '@/stores/Repo/praise_data';
import { useMrChangeStore } from '@/stores/merge';
import { AuthMap } from '@/constant';
import { useUploadFile } from '@/utils/hooks/useUploadFile';
import { i18n } from '@/i18n';
import { currentTheme } from '@/themes';
import { getLineRangeVisible, getLineRangeConfig } from '@/views/Repo/Merge/utils/handleMultiLine';
import { codeReviewOptions } from '@/constant/mr';
import { AuditStatusEnum } from '@/api/issue/types';
import { getUserValid } from '@/utils/getUserValid';
import Praise from '@/components/praise/index.vue';
import ContentVersionList from './ContentVersionList.vue';
import ContentDiffModal from './ContentDiffModal.vue';
import { GModal } from '@/components/Setting/index';
import { useTimeFormat } from '@/utils/hooks/useTimeFormat';
import { VITE_HOST } from '@/utils/tools';
import { REPO_ISSUE_CONTENT_MAX_LENGTH } from '@/constant/issue';

const { formatTimeWithMonthAndYear } = useTimeFormat();
const hostURL = VITE_HOST(); // 主站的域名
const commitFilePath = computed(() => {
  const commitHash = props.data?.position?.head_sha;
  return `${hostURL}/${props.pathWithNamespace}/blob/${(commitHash || (props.sourceBranch as string)).replace(/\//g, '%2F')}/${props.diffFile}`;
});
const { t } = i18n.global;
const { upload } = useUploadFile();
const router = useRouter();
const route = useRoute();
const { isArchived, isVisitorType, issueJumpCommentId } = storeToRefs(repoInfoStore());
const { accountInfo } = useAccountStore();

const props = withDefaults(
  defineProps<{
    type?: 'issue' | 'mr' | 'commit'; // ········类型
    author?: IAuthor; // ·········· 评论作者
    createdAt?: string; // ········ 创建时间
    data: any; // ·············· 评论数据对象
    eventType?: string; // ········ 事件类型
    eventIcon?: string; // ········ 事件icon
    eventMsg?: string; // ········· 事件信息
    noteNum?: number; // ·········· 评论数量
    targetId: number | string; // ··评论 note_id
    discussionId?: string; // ·······评论 discussion_id
    realIssueId?: string | number; // ········ realIssueId
    body: string; // ·············· 评论或描述body
    showBody?: string; // ·············· 评论或描述body（转换后）
    sourceBranch?: string; // ····· 源分支名
    canEdit?: boolean; // ··········是否-可编辑
    canQuoteReply?: boolean; // ····是否-可引用回复
    canDelete?: boolean; // ······· 是否-可删除评论
    moreMenu?: boolean; // ········ 展示更多操作
    loading?: boolean; // ········· 编辑加载状态
    repoId?: string; // ··········· 项目id
    issueId?: number; // ·········· issue id
    editMaxLength?: number; // ·····评论内容最大长度
    pathWithNamespace?: string; // ······ 项目路径
    discussionType?: COMMENT; // ·······issue描述或者pr描述
    riskLevel?: RiskLevel;
    autoComplete?: boolean; // 表情点赞功能是否自动加载
    labelList?: any[]; // 标签列表
    enterpriseLabelList?: any[]; // 企业标签列表
    enableIssueOrPrHint?: boolean; // 是否开启issue或pr快捷选择
    // 检视意见特有 props
    isReviewComment?: boolean; // ········· 是否为检视意见：包括代码检视意见+文件级检视意见
    isCodeReviewComment?: boolean; // ····是否是代码检视意见
    isFileReviewComment?: boolean; // ····是否是文件检视意见
    isReply?: boolean; // ········· 是一个解决问题的评论
    diff?: string; // ············· 具体文件变动内容
    diffFile?: string; // ········· 变动文件名称
    noteJumpInfo?: NoteJumpInfo; // 检视意见-目标文件-跳转信息
    addedLines?: number; // ······· 变动添加
    removedLines?: number; // ······变动移除
    resolvable?: boolean; // ······ 展示解决按钮
    resolved?: boolean; // ········ 是否已解决
    canResolve?: boolean; // ······ 可点击解决；禁用状态
    formatIng?: boolean; // ······· 转码状态
    replyDisabled?: boolean; // ····回复禁用
    replayMaxLength?: number; // ···可回复的文本长度
    isPrCodeDiff?: boolean; // 是否是pr检视意见评论-主体
  }>(),
  {
    editMaxLength: REPO_ISSUE_CONTENT_MAX_LENGTH,
    replayMaxLength: REPO_ISSUE_CONTENT_MAX_LENGTH,
    autoComplete: false,
    enableIssueOrPrHint: false,
    isPrCodeDiff: false
  }
);
const templateMdBody = ref('');
const mdBody = computed(() => {
  return templateMdBody.value || props.showBody || props.body;
});
const lineRangeVisible = computed(() => {
  return getLineRangeVisible(props?.data?.position);
});
const lineRangeConfig = computed(() => {
  return getLineRangeConfig(props?.data?.position);
});
const diffOptions = ref(codeReviewOptions);
const { jumpCommentId } = storeToRefs(useMrChangeStore());
// 显示角色
const role = computed(() => {
  return props.data?.role?.cn_name || '';
});

const showResolveSwitch = computed(() => {
  return props?.resolvable && !props?.isReply && (!props?.isCodeReviewComment || props?.isFileReviewComment) && !props?.data?.is_reply;
});

const discussion_id = computed(() => {
  // 描述需要传prid或者issueid
  return props?.discussionId || props?.realIssueId;
});
const praiseResourceId = computed(() => {
  if (props.discussionType === COMMENT.MR || props.discussionType === COMMENT.ISSUE) {
    return props.targetId;
  }
  return props?.discussionId || props?.realIssueId || props?.targetId;
});
const realDiscussion = ref(); // 真实评论
const eventIconName = computed(() => {
  if (props.eventType === 'commit') return 'gt-line-commit';
  if (props.eventType === 'discussion') return 'gt-line-discussions';
  if (props.eventType === 'associate_kanban' || props.eventType === 'remove_kanban') {
    return 'gt-line-table';
  }
  if (props.eventType === 'pin_issue') {
    return 'gt-line-Topping';
  }
  if (props.eventType === 'unpin_issue') {
    return 'gt-line-cancelTopping';
  }
  if (
    props.eventType === 'changed_issue_custom_state' ||
    props.eventType === 'changed_issue_custom_severity' ||
    props.eventType === 'changed_issue_custom_category' ||
    props.eventType === 'issue_custom_field'
  ) {
    return 'gt-line-issues';
  }
  return props.eventIcon;
});

const discussionAuthorName = computed(() => {
  return cAuthor.value ? pickNickName(cAuthor.value) : getUserValid();
});

const canEditDiscussion = computed(() => {
  return props.canEdit && !isArchived.value && !props.diff;
});

/** 仅展示时间 */
const onlyShowTime = computed(() => {
  if (props.eventType && [RiskLevel.INIT, RiskLevel.REJECT].includes(props?.riskLevel)) {
    // 审核时
    return false;
  } else if (props.createdAt && !props.eventMsg) {
    return true;
  } else {
    return false;
  }
});

/** 仅展示事件 */
const onlyShowEventMsg = computed(() => {
  if (props.eventType && [RiskLevel.INIT, RiskLevel.REJECT].includes(props?.riskLevel)) {
    // 审核时
    return true;
  } else if (props.createdAt && props.eventMsg && !props.diff) {
    return true;
  } else {
    return false;
  }
});

/* 是否含有action */
const hasAction = computed(() => {
  return props.data?.action;
});

const cAuthor = computed(() => {
  if (
    props.eventType &&
    typeof props.author?.audit_status === 'number' &&
    props.author.audit_status !== AuditStatusEnum.PASS
  ) {
    return {
      username: '',
      nickname: '匿名用户',
      avatar_url: ''
    };
  } else {
    return props.author;
  }
});

// 是否为评论作者(决定能不能修改评论中的checkbox)
const isAuthor = computed(() => {
  return cAuthor.value?.username === accountInfo.username;
});

const emit = defineEmits<{
  'on-remove': [obj: object];
  'quote-reply': [value: string];
  'save-content': [value: string];
  'reply-discussion': [value: string];
  'resolve-stat-change': [status: boolean];
  'apply-suggestion': [obj: object];
  'do-format': [];
}>();

const commitModelVisible = ref(false);
const commitContent = ref('');
const handleAcceptSuggestion = (evt) => {
  commitContent.value = decodeURIComponent(evt.target.nextSibling.value);
  commitModelVisible.value = true;
};

const isSuggestion = computed(() => {
  return props.data?.severity === 'suggestion';
});
const isExpiredSuggestion = computed(() => {
  return props.data?.is_expired_suggestion;
});

const suggestionExpand = ref(false);

const canDeleteIssue = computed(() => {
  return props.canDelete;
});

const isAcceptedSuggestion = computed(() => {
  const codeSnippet = '```suggestion \n' + props.data?.content + '\n```';
  return props.body && props.body?.indexOf(codeSnippet) >= 0;
});

const customParseInject = inject('customParse') as Function;
const lineCodeInject = inject('lineCode');
const customParse = (html: string, accept?: boolean) => {
  if (customParseInject && props.isReviewComment) {
    return customParseInject(html, accept, {
      isExpiredSuggestion: isExpiredSuggestion.value,
      changeContent: props.data?.content
    });
  }
  return html;
};

// 是否为成员（评论中显示成员身份, 至少为浏览者）
const isMember = computed(() => {
  const authorIsVisitor = accountInfo.username === props.author?.username && isVisitorType.value; // 根据当前用户权限判断是否为成员
  const authorIsMember = props.author?.is_member || props.author?.isMember; // 接口返回是否为成员
  return authorIsVisitor || authorIsMember;
});

const tid = computed(() => (props.targetId ? 'tid-' + props.targetId : ''));
const deleteModalVisible = ref(false);
const renderModal = ref(false);
const isFold = ref(false);
const editText = ref('');
const editIng = ref(false);
const RefMd = ref();
const pageFrom = inject('pageFrom');
const originalComment = ref();
const pageFromValue = computed(() => {
  if (props.discussionType) {
    return props.discussionType;
  }
  switch (pageFrom) {
    case 'issue':
      return COMMENT.ISSUE;
    case 'mr':
      return COMMENT.MR;
    case 'commit':
      return COMMENT.COMMIT;
    default:
      return '';
  }
});

const replyLoading = ref(false);
const replyIng = ref(false);
const replyText = ref('');
const RefReplyMD = ref();
const delIng = ref(false);
const isResolved = ref(props.resolved);

const iid = inject('iid');
const getGroupMembers = inject('getGroupMembers') as () => Promise<void>;

import { createListFetchers } from '@/components/IssueOrPrItemMini/index';
const { getIssueList, getMergeList } = createListFetchers(props.repoId as string);
const hintConfig = computed(() => {
  const config: Record<string, (params: { search?: string }) => Promise<any>> = {
    '@': getGroupMembers
  };

  if (props.enableIssueOrPrHint) {
    config['#'] = getIssueList;
    config['!'] = getMergeList;
  }
  return config;
});

const handleEdit = async () => {
  const searchDiscussionId = props.isReply ? discussion_id.value + '_' + props.targetId : discussion_id.value;
  const discussRes = await getOriginalComment({ type: pageFromValue.value, discussionId: searchDiscussionId });
  editText.value = discussRes?.data.data?.origin_discussion || (props.body || '').substring(0);
  realDiscussion.value = editText.value; // 保存一下原始评论
  editIng.value = true;
};
watch(
  () => props.body,
  () => {
    editIng.value = false;
    editText.value = '';
  }
);
watch(
  () => props?.resolved,
  (value) => {
    console.log('resolved', value);
    isResolved.value = value;
  }
);

// 复制
const { copy } = useClipboard({ source: 'text', legacy: true });
const onCopy = () => {
  // 需要拼接分支名称
  const refBranch = route.query?.ref || '';
  if (discussion_id.value) {
    copy(location.origin + location.pathname + `?ref=${refBranch}` + '&did=' + discussion_id.value + '#' + tid.value);
  } else {
    copy(location.origin + location.pathname + `?ref=${refBranch}` + '#' + tid.value);
  }
  DMessage.success(t('common.msg.copedLink'));
  document.body.click();
};

const copyTip = ref(t('common.copy'));
let timer: any = null;
const onCopyPath = () => {
  copy(props.diffFile);

  copyTip.value = t('common.tips.copied');
  timer && clearTimeout(timer);
  timer = setTimeout(() => (copyTip.value = t('common.copy')), 2000);
};
// 聚焦到 回复-编辑器 最后
const focusReplyMdEditor = async (_cursorPosition: number = 0) => {
  await nextTick();
  const lastLineNumber = RefReplyMD.value.instance.lastLine();
  RefReplyMD.value.instance.setCursor(lastLineNumber, 0);
  RefReplyMD.value.instance.scrollIntoView(null, RefReplyMD.value.instance.getScrollInfo().clientHeight - 10);
  RefReplyMD.value.instance.focus();
};

const setReply = (content: string) => {
  replyIng.value = true;
  replyText.value = content;
};

const quoteReply = async () => {
  document.body.click();
  const discussRes = await getOriginalComment({ type: pageFromValue.value as COMMENT, discussionId: discussion_id.value });
  let content: string;
  if (props.isReply && props.type === 'mr') {
    content = props.body;
  } else {
    content = discussRes?.data?.data?.origin_discussion || props.body;
  }
  if (props.isReviewComment) {
    // 检视意见引用回复时，触发的是”评论-回复“操作，而不是新评论
    const body = formatQuoteReply(content);
    const atCommentAuthor = formatAtAuthor(props.author?.username);
    replyText.value = body + atCommentAuthor;
    replyIng.value = true;
    // setTimeout(() => {
    //   focusReplyMdEditor(atCommentAuthor.length);
    // }, 100);
    return;
  } else {
    // 触发的是新评论
    emit('quote-reply', content);
  }
};

const handleReplyComment = () => {
  replyLoading.value = true;
  emit('reply-discussion', replyText.value);
};
const onBeforeDelete = async () => {
  document.body.click();
  if (props.noteNum && props.noteNum > 1) {
    DMessage.info(t('repo.issue.cannotDelete'));
    return;
  }
  const discussRes = await getOriginalComment({ type: pageFromValue.value, discussionId: discussion_id.value });
  originalComment.value = discussRes?.data.data?.origin_discussion || (props.body || '').substring(0);
  nextTick(() => { deleteModalVisible.value = true; });
};
const handleExpand = () => {
  isFold.value = false;
  fullscreen('code-review-' + props.diffFile);
};

const handleSuggestionExpand = () => {
  suggestionExpand.value = !suggestionExpand.value;
};
const confirmDelete = () => {
  emit('on-remove', props.data);
  delIng.value = true;
};

const reset = () => {
  editText.value = '';
  editIng.value = false;

  replyText.value = '';
  replyLoading.value = false;
  replyIng.value = false;
};

const focusEditor = async (_cursorPosition: number = 0) => {
  await nextTick();
  const lastLineNumber = RefMd.value.instance.lastLine();
  RefMd.value.instance.setCursor(lastLineNumber, 0);
  RefMd.value.instance.scrollIntoView(null, RefMd.value.instance.getScrollInfo().clientHeight - 10);
  RefMd.value.instance.focus();
};

const handleViewSource = () => {
  if (props.pathWithNamespace) {
    toOriginWeb();
    return false;
  }
  const defaultBranch = props.sourceBranch;
  const branchName = Array.isArray(defaultBranch) ? defaultBranch : defaultBranch.split('/');
  router.push({
    name: 'repoFile',
    params: {
      branchName: branchName,
      filePath: props.diffFile
    }
  });
};

// 应用建议成功，刷新界面
const onSuccessCommit = () => {
  emit('apply-suggestion', props.data);
};

// 查看源码跳转
const toOriginWeb = () => {
  window.open(commitFilePath.value, '_blank');
};

const uploading = ref(false);
const attachmentType = inject('attachmentType');
const showFileUpload = inject('canUpload');

const handleFileUpload = () => {
  if (uploading.value) return;
  RefMd.value.fileUploadHandler();
};
const fileUpload = async (file, callback) => {
  const { name, size, type } = file;
  const uploadType = attachmentType;
  const params = {
    size,
    file_name: name,
    content_type: type,
    type: uploadType, // ISSUE, PR
    project_id: encodeURIComponent(props.repoId)
  };
  uploading.value = true;
  const fileInfo = await upload(file, params);
  if (fileInfo) {
    callback(fileInfo);
    uploading.value = false;
    return true;
  }
  uploading.value = false;
};

let bodyChangeTimer: any = null;

const saveContent = (html: string) => {
  // 送去外部保存的时候清空临时存储内容。
  // templateMdBody.value = '';
  clearTimeout(bodyChangeTimer);
  bodyChangeTimer = setTimeout(() => {
    emit('save-content', html);
  }, 0);
};

const changeContentChecked = (html: string) => {
  if (props.loading) return;
  if (canEditDiscussion.value) {
    // 立即修改显示内容，防止连续点击的时候，下一次没有给回最新的内容
    // 对比 html 和 props.body，同步 checkbox 状态
    const syncCheckboxState = (htmlContent: string, markdownBody: string): string => {
      console.log('htmlContent', htmlContent);
      console.log('markdownBody', markdownBody);
      try {
        // 方法1: 尝试从 HTML 中提取 checkbox 标签
        const checkboxRegex = /<input[^>]*type=["']checkbox["'][^>]*>/gi;
        const checkboxMatches = htmlContent.match(checkboxRegex) || [];
        const checkboxStates: Array<{ checked: boolean; index: number }> = [];
        if (checkboxMatches.length > 0) {
          // 从 HTML 标签中提取状态
          checkboxMatches.forEach((match, index) => {
            const hasChecked = /checked(?:\s*=\s*["']?(?:checked|true)["']?)?/i.test(match);
            checkboxStates.push({
              checked: hasChecked,
              index
            });
            console.log(`Checkbox ${index}: HTML标签=${match}, checked=${hasChecked}`);
          });
        } else {
          // 方法2: 如果找不到 HTML 标签，尝试从 HTML 内容中提取 markdown 格式的 checkbox
          // 这可能是 HTML 中包含了 markdown 文本的情况
          // 匹配格式：1. 缩进 + 列表标记符(-*+，可多个) + 空格 + [ ]/[x]/[X] + 文本
          //          2. 缩进 + [ ]/[x]/[X] + 文本（行首直接出现）
          const markdownCheckboxRegex = /^(\s*)(?:[-*+](?:\s+[-*+])*\s+)?(\[[\sxX]\])\s*(.*)$/gm;
          const markdownMatches = htmlContent.match(markdownCheckboxRegex) || [];
          if (markdownMatches.length > 0) {
            markdownMatches.forEach((match, index) => {
              const checkboxMatch = match.match(/\[([\sxX])\]/);
              if (checkboxMatch) {
                const isChecked = checkboxMatch[1].toLowerCase() === 'x';
                checkboxStates.push({
                  checked: isChecked,
                  index
                });
                console.log(`Checkbox ${index}: Markdown格式=${match}, checked=${isChecked}`);
              }
            });
          }
        }
        console.log(`找到 ${checkboxStates.length} 个 checkbox，状态:`, checkboxStates.map(s => s.checked));
        // 如果没有 checkbox，直接返回原 markdown
        if (checkboxStates.length === 0) {
          console.log('没有找到 checkbox，返回原 markdown');
          return markdownBody;
        }
        // 解析 markdown，找到所有 checkbox 行并更新
        const lines = markdownBody.split('\n');
        let checkboxIndex = 0;
        const updatedLines = lines.map((line, lineIndex) => {
          // 匹配行首的 checkbox 格式：
          // 1. 缩进 + 列表标记符(-*+，可多个) + 空格 + [ ]/[x]/[X] + 文本
          // 2. 缩进 + [ ]/[x]/[X] + 文本（行首直接出现）
          const checkboxMatch = line.match(/^(\s*)(?:[-*+](?:\s+[-*+])*\s+)?(\[[\sxX]\])\s*(.*)$/);
          if (checkboxMatch) {
            if (checkboxIndex < checkboxStates.length) {
              const state = checkboxStates[checkboxIndex];
              const indent = checkboxMatch[1];
              const content = checkboxMatch[3];
              // 根据 HTML 中的状态更新 markdown checkbox
              const newCheckbox = state.checked ? '[x]' : '[ ]';
              // 检查原行是否有列表标记符，如果有则保持原有格式
              const listMarkerMatch = line.match(/^(\s*)([-*+](?:\s+[-*+])*)\s+(\[[\sxX]\])\s*(.*)$/);
              let newLine;
              if (listMarkerMatch) {
                // 有列表标记符：保持格式 "缩进 + 列表标记符 + 空格 + 复选框 + 空格 + 内容"
                newLine = `${indent}${listMarkerMatch[2]} ${newCheckbox} ${content}`;
              } else {
                // 没有列表标记符：格式 "缩进 + 复选框 + 空格 + 内容"
                newLine = `${indent}${newCheckbox} ${content}`;
              }
              console.log(`行 ${lineIndex}: 原="${line}", 新="${newLine}", HTML状态=${state.checked}`);
              checkboxIndex++;
              return newLine;
            } else {
              console.warn(`行 ${lineIndex}: 找到 checkbox 但 HTML 中没有对应的状态，保持原样`);
            }
          }
          return line;
        });
        if (checkboxIndex !== checkboxStates.length) {
          console.warn(`警告: markdown 中找到 ${checkboxIndex} 个 checkbox，但 HTML 中有 ${checkboxStates.length} 个`);
        }
        return updatedLines.join('\n');
      } catch (error) {
        console.error('同步 checkbox 状态失败:', error);
        return markdownBody;
      }
    };
    const updatedBody = syncCheckboxState(html, props.body);
    console.log('更新后的 body:', updatedBody);
    // 将更新后的 markdown 存储到 templateMdBody 用于显示
    templateMdBody.value = updatedBody;
    // 保存更新后的 markdown body，而不是 HTML
    saveContent(updatedBody);
  }
};

const handleDiffFileClick = () => {
  useMrChangeStore().setDiffFileJumping(true);
};

// ------------------ 评论点赞功能 ---------------------------//
const { discussionPraiseData } = storeToRefs(usePraiseDataStore());
const discussPraiseEmojiList = ref<any[]>([]);

watch(
  [() => discussionPraiseData.value, praiseResourceId],
  ([newVal, resourceId]) => {
    if (props.discussionType !== COMMENT.MR && props.discussionType !== COMMENT.ISSUE) return;
    if (!resourceId) return;
    if (!Array.isArray(newVal) || newVal.length === 0) {
      discussPraiseEmojiList.value = [];
      return;
    }

    const temp = newVal.find((item) => item.resource_id === resourceId.toString());
    discussPraiseEmojiList.value = temp?.emoji_counts ?? [];
  },
  { immediate: true }
);

// ---------------- 评论历史版本功能 -------------------------//
import { getEditVersionForDesc } from '@/utils/hooks/useEditVersion';
import { editVersionDataStore } from '@/stores/Repo/edit_version';

// 是否展示历史修改入口
const { editVersionData } = storeToRefs(editVersionDataStore());
const showEditVersion = ref(false); // 是否展示历史修改入口

// 单查 issue/pr 描述的历史版本记录
if (props.autoComplete) {
  const resource_id = props?.discussionId || props?.realIssueId || props?.targetId;
  getEditVersionForDesc(resource_id, props.discussionType);
}

watch(
  [() => editVersionData.value, praiseResourceId],
  ([newVal, resourceId]) => {
    if (![COMMENT.ISSUE, COMMENT.MR, COMMENT.ISSUE_DESC, COMMENT.PR_DESC].includes(props.discussionType)) return;
    if (!resourceId) return;
    if (!Array.isArray(newVal) || newVal.length === 0) {
      showEditVersion.value = false;
      return;
    }

    const temp = newVal.find((item) => item.resource_id === resourceId.toString());
    showEditVersion.value = temp?.exist_audit_log || false;
  },
  { immediate: true }
);

// 评论 header 中，是否展示历史修改记录入口
const showVersionDropdown = computed(() => {
  if (hasAction.value) return false;
  // 仅处理：issue/pr 评论+描述
  if (![COMMENT.ISSUE, COMMENT.MR, COMMENT.ISSUE_DESC, COMMENT.PR_DESC].includes(props.discussionType)) return false;
  if (editIng.value) return false;
  return showEditVersion.value;
});

// 历史修改-UI交互
const versionToggleStatus = ref(false);
const versionDropdownVisible = ref(false);

// 权限-是否可删除版本
const canVersionDelete = computed(() => {
  if (props.discussionType === COMMENT.ISSUE_DESC || props.discussionType === COMMENT.ISSUE) {
    return props.canEdit;
  }
  if (props.discussionType === COMMENT.PR_DESC) {
    return canEditDiscussion.value;
  }
  if (props.discussionType === COMMENT.MR) {
    // 检视意见
    if (props.diff) {
      return props.canEdit && !isArchived.value;
    }
    return canEditDiscussion.value;
  }
  return false;
});
// 历史修改-diff 对比弹窗交互
const diffModalVisible = ref(false);
const diffModalData = ref({
  currentItem: null,
  previousItem: null
});
import { IVersion } from './contentVersionType';
const handleVersionDiffView = (data: { currentItem: IVersion, previousItem: IVersion }) => {
  diffModalData.value = data;
  diffModalVisible.value = true;
};
const delVersionVisible = ref(false);
const versionDelIng = ref(false);
const delVersionInfo = ref({
  id: '',
  canDelete: false,
  title: '',
  tip01: '',
  tip02: ''
});
const handleDeleteVersion = (data: { id: string, is_latest: boolean; }) => {
  if (!canVersionDelete.value) return;
  const { id, is_latest } = data;
  if (is_latest) {
    delVersionInfo.value = {
      id,
      canDelete: false,
      title: t('repo.issueVersion.noDeleteTip01'),
      tip01: t('repo.issueVersion.noDeleteTip02'),
      tip02: t('repo.issueVersion.noDeleteTip03')
    };
  } else {
    delVersionInfo.value = {
      id,
      canDelete: true,
      title: t('common.delete'),
      tip01: t('repo.issueVersion.deleteTip01'),
      tip02: t('repo.issueVersion.deleteTip02')
    };
  }
  delVersionVisible.value = true;
};

const confirmVersionDelete = async () => {
  versionDelIng.value = true;
  const res = await deleteVersion({
    project_id: props.repoId,
    id: delVersionInfo.value.id,
    type: props.discussionType
  });
  if (res.status === 200) {
    DMessage.info(t('common.msg.deleted'));
    diffModalVisible.value = false;
    versionDropdownVisible.value = false;
  }
  versionDelIng.value = false;
  delVersionVisible.value = false;
};

defineExpose({
  reset,
  setReply,
  focusEditor,
  focusReplyMdEditor
});
</script>
<style lang="scss" scoped>
.bottom-line::after {
  content: '';
  height: 32px;
  width: 1px;
  background: var(--theme-line);
  position: absolute;
  left: 27px;
}

.bottom-line:last-of-type::after {
  background: none;
}

.bottom-line::before {
  content: '';
  height: 32px;
  width: 1px;
  background: var(--theme-line);
  position: absolute;
  top: -32px;
  left: 27px;
}

.bottom-line:first-of-type::before {
  background: none;
}

.g-discussion-item {
  position: relative;
  line-height: 20px;
  border-top: 1px solid transparent;
  border-left: 1px solid transparent;
  border-right: 1px solid transparent;
  border-bottom: 1px solid var(--theme-dividing-line, #f0f1f2);
  @apply bg-[var(--theme-default-bg)] text-sm;

  &:first-of-type {
    border-top-left-radius: var(--border-radius);
    border-top-right-radius: var(--border-radius);
  }

  &:last-of-type {
    border-bottom-left-radius: var(--border-radius);
    border-bottom-right-radius: var(--border-radius);
    border-bottom: 1px solid transparent;
  }

  &.active {
    border: 1px solid var(--color-primary);
    border-radius: 2px;
  }

  :deep(.comment-icon) {
    display: none !important;
  }

  :deep(.devui-code-review__header) {
    display: none !important;
  }
}

.diff-header {
  background: var(--devui-global-bg);
  border-bottom: 1px solid var(--theme-line-border);
  border-top: 1px solid var(--theme-line-border);
  scroll-margin-top: 160px;
}

.g-content-card-v2 {
  scroll-margin-top: 160px;

  :deep(.devui-code-review__header) {
    display: none !important;
  }
}

.g-discussion-header.g-discussion-header-only-event-msg {
  @apply px-[16px] py-[0px];
}

.g-discussion-header {
  @apply px-[16px] py-[12px];

  .comment-info {
    @apply flex gap-[8px] flex-col md:flex-row items-start whitespace-nowrap overflow-hidden text-ellipsis text-[var(--theme-text)];
  }

  .g-discussion-status {
    @apply w-[24px] min-w-[24px] h-[24px] flex items-center justify-center bg-[var(--theme-global-bg)];
    border-radius: 12px;
  }

  .g-discussion-header-info {
    font-size: 14px;
    @apply flex-grow flex gap-2 items-start break-all text-ellipsis overflow-hidden items-center;

    .header-info {
      @apply flex-1 whitespace-nowrap overflow-hidden text-ellipsis;
    }

    .avatar-link {
      @apply min-w-[20px] inline-flex items-center;
    }

    .user-link {
      &:hover {
        text-decoration: underline;
      }

      @apply min-w-[1em] overflow-hidden text-ellipsis max-w-[240px];
    }

    @media screen and (min-width: $screen-lg) {
      .user-link {
        @apply max-w-[240px];
      }
    }

    @media screen and (min-width: $screen-xl) {
      .user-link {
        @apply max-w-[260px];
      }
    }

    @media screen and (min-width: 1980px) {
      .user-link {
        @apply max-w-[300px];
      }
    }

    .btn {
      @apply cursor-pointer active:bg-CG300 rounded-sm;
    }

    .code-review-switch {
      @apply min-w-[20px] h-[16px] w-[20px] p-0 text-center rounded cursor-pointer active:bg-[var(--theme-bg-hover)];
    }

    .file-change-info-group {
      @apply inline-flex items-center gap-2 overflow-hidden text-ellipsis;

      .diff-file-name {
        @apply text-ellipsis overflow-hidden;

        &:hover {
          color: var(--theme-link);
          text-decoration: underline;
          cursor: pointer;
        }
      }

      :deep(.icon) {
        @apply min-w-[16px];
      }
    }
  }

  .g-discussion-header-right {
    height: 20px;
    @apply inline-flex items-center;

    .resolved-group {
      @apply flex items-center gap-2 whitespace-nowrap;

      .ques-switch {
        color: var(--devui-aide-text);
      }
    }
  }
}

.event-msg-wrap {
  @apply flex-1;
  color: var(--devui-placeholder);
  @apply min-w-[1em] overflow-hidden text-ellipsis;

  @media screen and (max-width: $screen-md) {
    white-space: normal;

    :deep(.g-fold-ellipsis-text) {
      width: auto !important;
    }

    :deep(.g-fold-ellipsis-switch) {
      display: none;
    }
  }
}

.g-discussion-permission {
  margin-top: 2px;
  min-width: 34px;
  border-radius: 4px;
  padding: 4px;
  background-color: var(--theme-badge-bg);
  text-align: center;
  font-size: 12px;
  font-weight: 500;
  line-height: 12px;
  color: var(--theme-aide-text);
}

:deep(.devui-switch__wrapper) {
  background: var(--checkbox-border-color, #d1d1e0);
}

.create-time {
  color: var(--theme-placeholder);
}

.g-discussion-divide-line {
  border-bottom: 1px solid var(--theme-line-border);
}

.g-discussion-body {
  position: relative;

  :deep(.gitCode-md) {
    overflow-y: hidden;
    overflow-x: auto;
  }

  .edit-area {
    @apply flex items-center gap-4 mt-4 mb-4;

    @media screen and (max-width: 500px) {
      flex-wrap: wrap;

      .edit-area_actions {
        flex-wrap: wrap;
        flex-direction: row;
      }
    }
  }
}

// 子评论
.outside~.g-discussion-item {
  @apply border-transparent;
  border-top: 1px solid var(--theme-dividing-line);
}

.outside~.g-discussion-item.active {
  border: 1px solid var(--theme-line-border);
  border-radius: 2px;
}

.outside {
  // @apply border border-transparent;
  background: var(--theme-default-bg);
  border-radius: 0 0 4px 4px;
}

.outside.active {
  border: 1px solid var(--theme-line-border);
  border-radius: 2px;
}

// 回复
.discussion-reply-group {
  border-top: 1px solid var(--theme-line-border);
  padding: 8px 20px 8px 56px;

  .discussion-reply-action {
    margin-top: 8px;
    @apply flex items-center gap-4 flex-row-reverse;
  }
}

.code-review-box {
  background-color: var(--theme-default-bg);

  :deep(.d2h-code-linenumber) {
    cursor: default;
  }
}

.g-only-event-msg {}

.jump-comment-focus.g-discussion-body {
  border: 1px solid var(--theme-info);
}

.content-version-dropdown {
  color: var(--theme-text);
  font-weight: 400;
  font-size: 14px;
  @apply flex items-center overflow-hidden whitespace-nowrap text-ellipsis;
}
</style>
<style lang="scss">
.check-box-disabled input[type='checkbox'] {
  opacity: 0.5;
  cursor: not-allowed;
  pointer-events: none;
}

.check-box-disabled label {
  cursor: not-allowed;
  pointer-events: none;
}

.devui-modal__overlay:has(+ .version-del-modal) {
  // 版本删除弹窗的 mask，要在详情 diff 弹窗层级上
  z-index: var(--devui-z-index-modal, 1050) !important;
}
</style>
