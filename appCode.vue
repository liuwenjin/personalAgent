<template>
  <div class="chatbox-bg tw-flex tw-flex-col tw-w-full tw-h-full" ref="rootComponentContainer" data-edit-id="1">
    <!-- ======================= 导航栏 ======================= -->
    <header v-if="(isSolo || content.isSolo) && dataEditable" class="chatbox-navbar" data-edit-id="2">
      <div class="navbar-inner" data-edit-id="3">
        <div class="navbar-left" data-edit-id="4">
          <div class="navbar-logo" :style="{ background: 'var(--muse-primary)' }" data-edit-id="5">
            <span class="navbar-logo-text" data-edit-id="6">{{
              icon || "✦"
            }}</span>
          </div>
          <div class="navbar-title" data-edit-id="7">
            <span class="navbar-title-text" data-edit-id="8">{{ title }}</span>
            <span class="navbar-introduce" data-edit-id="9">{{
              introduce
            }}</span>
          </div>
        </div>
				<div class="tw-flex tw-items-center" data-edit-id="103">
        <slot v-if="step === 0 && !flag" data-edit-id="115">
          <el-button class="wizard-action-btn" @click="openFeedbackDialog()" data-edit-id="116">
            问题反馈
          </el-button>
        </slot>
        <div appName="loginRegItem" component="CardItem" class="tw-opacity-100 refined-user-card" :style="{
            width: '135px',
            marginLeft: '15px',
            height: '30px',
            borderLeft: 'solid 1px #ddd',
          }" data-edit-id="104"></div>
      </div>
      </div>
      
    </header>

    <!-- ======================= 消息区 ======================= -->
    <main class="chatbox-main" ref="scrollContainer" @scroll="onScroll" data-edit-id="14">
      <!-- 空状态 -->
      <div v-if="messages.length === 0" class="chatbox-empty" data-edit-id="15">
        <div class="empty-rune" data-edit-id="16">
          <div class="empty-ring" data-edit-id="17"></div>
          <i class="bi bi-chat-square-text empty-icon" data-edit-id="18"></i>
        </div>
        <h2 class="empty-title" data-edit-id="19">
          {{ title || "你好，我是你的助手" }}
        </h2>
        <p class="empty-subtitle" data-edit-id="20">
          {{ introduce || "有什么可以帮你的？" }}
        </p>

        <div class="empty-chips" v-if="exampleList && exampleList.length" data-edit-id="21">
          <button v-for="(d, i) in exampleList" :key="i" class="empty-chip" @click="useExample(d)" data-edit-id="22">
            <span class="chip-icon" data-edit-id="23">{{ d.icon || "✦" }}</span>
            <span data-edit-id="24">{{ d.name || d.title }}</span>
          </button>
        </div>
      </div>

      <!-- 消息列表 -->
      <div v-else class="message-list" data-edit-id="25">
        <article v-for="m in messages" :key="m.id" class="message-row" :class="
            m.role === 'user' ? 'message-row--user' : 'message-row--assistant'
          " data-edit-id="26">
          <div class="message-body" data-edit-id="29">
            <div class="bubble" :class="m.role === 'user' ? 'bubble--user' : 'bubble--assistant'" data-edit-id="33">
              <!-- 流式中的光标 -->
              <template v-if="m.role === 'assistant' && !m.content && m.streaming">
                <span class="typing-dots"> <i></i><i></i><i></i> </span>
              </template>
              <!-- 已渲染内容 -->
              <div v-else class="bubble-content" v-html="renderMarkdown(m.content)"></div>
              <span v-if="m.role === 'assistant' && m.streaming && m.content" class="stream-cursor"></span>
            </div>

            <!-- 助手消息操作：可配置的动态按钮列表 / 下拉菜单 / 组合输入框 -->
            <div v-if="
                m.role === 'assistant' &&
                m.content &&
                effectiveMessageActions.length
              " class="message-actions">
              <div v-for="(item, idx) in effectiveMessageActions" :key="idx">
                <!-- 分隔线 -->
                <span v-if="item.type === 'divider'" class="action-divider"></span>

                <!-- 组合输入框 + 按钮 -->
                <div v-else-if="isComboItem(item)" class="action-input-group">
                  <input class="action-input" :type="item.inputType || 'text'" :placeholder="item.placeholder || '输入内容…'" :maxlength="item.maxLength || undefined" :disabled="!!item.disabled" :title="item.tooltip" :data-key="item.key || ''" @input="onActionInput(m, item, $event)" @keydown.enter.stop.prevent="submitCombo(m, item, $event)" clearable />
                  <button type="button" class="mini-btn mini-btn--combo" :disabled="!!item.disabled" :title="item.buttonTooltip || item.tooltip" @click="submitCombo(m, item, $event)">
                    <i class="bi bi-chevron-right"></i>
                  </button>
                </div>

                <!-- 下拉菜单 -->
                <div v-else-if="item.type === 'dropdown'" class="action-dropdown" :class="{ open: isMenuOpen(m, item) }">
                  <button type="button" class="mini-btn" :disabled="!!item.disabled" :title="item.tooltip" @click.stop="toggleActionMenu(m, item)">
                    <i v-if="item.icon" :class="item.icon"></i>
                    <span>{{ item.label || "更多" }}</span>
                    <i class="bi bi-chevron-down action-arrow"></i>
                  </button>
                  <transition name="action-menu">
                    <div v-if="isMenuOpen(m, item)" class="action-menu">
                      <button v-for="child in item.children || []" :key="child.key || child.label" type="button" class="action-menu-item" :disabled="!!child.disabled" :title="child.tooltip" @click.stop="selectActionChild(m, item, child)">
                        <i v-if="child.icon" :class="child.icon"></i>
                        <span class="action-menu-label">{{ child.label }}</span>
                      </button>
                    </div>
                  </transition>
                </div>

                <!-- 普通按钮（默认类型） -->
                <button v-else type="button" class="mini-btn" :class="{ 'mini-btn--active': !!item.active }" :disabled="!!item.disabled" :title="item.tooltip" :data-key="item.key || ''" @click="
                    dispatchMessageAction({
                      type: 'click',
                      key: item.key,
                      value: item.value,
                      item: item,
                      message: m,
                    })
                  ">
                  <i v-if="item.icon" :class="item.icon"></i>
                  <span>{{ item.label || "操作" }}</span>
                </button>
              </div>
            </div>
          </div>
        </article>
      </div>

      <!-- 回到底部 -->
      <transition name="fade-up">
        <button v-if="showScrollHint" class="scroll-bottom-btn" @click="scrollToBottom(true)">
          <i class="bi bi-arrow-down"></i>
        </button>
      </transition>
    </main>

    <!-- ======================= 子窗口层（同源内嵌 / 支持最小化 / 索引导航 / 可收起） ======================= -->
    <transition name="child-win">
      <section v-if="childWindows.length" v-show="childPanelVisible" class="child-window-layer" data-edit-id="subwin" style="
          position: absolute;
          right: 20px;
          bottom: 168px;
          width: min(740px, 64vw);
          height: min(62vh, 620px);
          min-height: 240px;
          z-index: 50;
          overflow: hidden;
        ">
        <!-- 索引导航（标签栏）：仅展示当前激活窗口，其余窗口自动最小化 -->
        <div class="child-window-nav" data-edit-id="subwin-nav" style="
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            height: 42px;
            box-sizing: border-box;
          ">
          <span class="child-nav-count" title="当前子窗口任务数">
            <i class="bi bi-window-stack"></i>
            <span>{{ childWindows.length }}</span>
          </span>
          <div class="child-nav-tabs">
            <button v-for="(win, idx) in childWindows" :key="win.id" type="button" class="child-nav-tab" :class="{
                active: win.id === activeChildId && !win.minimized,
                minimized: win.minimized,
              }" :title="(win.minimized ? '（已最小化）' : '') + win.title" @click="activateChildWindow(win.id)">
              <span class="child-nav-index">{{ idx + 1 }}</span>
              <span class="child-nav-title">{{ win.title }}</span>
              <i v-if="win.minimized" class="bi bi-window-stack child-nav-mini"></i>
              <span class="child-nav-close" title="关闭该窗口" @click.stop="closeChildWindow(win.id)">×</span>
            </button>
          </div>
          <div class="child-nav-actions">
            <button type="button" class="child-nav-btn" title="收起面板（保留任务，按钮恢复显示）" @click="toggleChildPanel">
              <i class="bi bi-chevron-down"></i>
            </button>
            <button type="button" class="child-nav-btn child-nav-btn--danger" title="关闭当前窗口" :disabled="!activeChildWindow" @click="closeChildWindow(activeChildId)">
              <i class="bi bi-x-lg"></i>
            </button>
          </div>
        </div>

        <!-- 子窗口内容：同一时刻仅激活且未最小化的窗口可见，其余保持挂载以保留状态 -->
        <div v-for="win in childWindows" v-show="win.id === activeChildId && !win.minimized" :key="win.id" class="child-window-body" style="
            position: absolute !important;
            top: 42px !important;
            left: 0 !important;
            right: 0 !important;
            bottom: 0 !important;
            overflow: hidden !important;
            background: #fff !important;
          ">
          <iframe :src="win.url" class="child-window-frame" style="
              position: absolute !important;
              top: 0 !important;
              left: 0 !important;
              right: 0 !important;
              bottom: 0 !important;
              width: 100% !important;
              height: 100% !important;
              border: none !important;
              background: #fff !important;
            " frameborder="0" allowfullscreen sandbox="allow-scripts allow-same-origin allow-forms allow-popups allow-modals allow-downloads"></iframe>
        </div>

        <!-- 全部最小化时的占位提示 -->
        <div v-if="!activeChildWindow || activeChildWindow.minimized" class="child-window-collapsed" style="position: absolute; top: 42px; left: 0; right: 0; bottom: 0">
          <i class="bi bi-window-stack child-collapsed-icon"></i>
          <span>所有子窗口已最小化，点击上方索引恢复</span>
        </div>
      </section>
    </transition>

    <!-- ======================= 输入区 ======================= -->
    <footer class="chatbox-composer">
      <div class="composer-inner">
        <!-- 子窗口控制按钮：固定在输入框右上方，常显任务数量，点击显示 / 隐藏浮窗 -->
        <button v-if="childWindows.length" type="button" class="child-trigger-btn" :class="{ 'child-trigger-btn--hidden': !childPanelVisible }" :title="
            childPanelVisible
              ? '收起子窗口面板（' + childWindows.length + ' 个任务）'
              : '显示子窗口面板（' + childWindows.length + ' 个任务）'
          " @click="toggleChildPanel">
          <i class="bi bi-window-stack"></i>
          <span class="child-trigger-text">子窗口</span>
          <span class="child-trigger-count">{{ childWindows.length }}</span>
        </button>
        <textarea ref="textarea" v-model="draft" class="composer-input" :placeholder="inputPlaceholder" rows="1" @input="autoResize" @keydown="onKeydown"></textarea>

        <div class="composer-footer">
          <div class="composer-hint">
            <span v-if="isStreaming" class="streaming-hint">
              <span class="hint-dot"></span>
              {{ loadingText }}
            </span>
            <template v-else>
              <span class="model-hint">
                <i class="bi bi-command"></i>
                {{ documentMeta.model }}/{{ documentMeta.apiName }}
              </span>
              <span v-if="!currentKnowledgeBase && !isStreaming" class="model-hint" title="自动决策：发送问题时根据问题内容自动选择知识库">
                <i class="bi bi-magic"></i>
                知识库自动决策
              </span>
              <button v-if="currentKnowledgeBase" type="button" class="kb-hint" :title="
                  '知识库：' +
                  currentKnowledgeBase.name +
                  '（' +
                  currentKnowledgeBase.id +
                  '）'
                " @click="toggleKbMenu">
                <i class="bi bi-journal-bookmark-fill"></i>
                <span>{{ currentKnowledgeBase.name }}</span>
              </button>
            </template>
          </div>

          <div class="composer-actions">
            <button v-if="isStreaming" class="send-btn send-btn--stop" @click="stopStreaming">
              <i class="bi bi-stop-fill"></i>
              <span>停止</span>
            </button>
            <button v-else class="send-btn" :disabled="!canSend" @click="send">
              <i class="bi bi-send-fill"></i>
              <span>发送</span>
            </button>
          </div>
        </div>
      </div>
    </footer>
  </div>
</template>

<style scoped>
  /* ============================================
   DESIGN TOKENS — Muse · 极简 · 精致
   ============================================ */
  * {
    --muse-primary: #0066cc;
    --muse-primary-hover: #0052a3;
    --muse-primary-active: #004c99;
    --muse-primary-soft: rgba(0, 102, 204, 0.08);
    --muse-primary-ring: rgba(0, 102, 204, 0.16);

    --muse-bg: #eef3f0;
    --muse-card: #f6f9f7;

    --muse-text: #0f172a;
    --muse-text-secondary: #475569;
    --muse-text-tertiary: #94a3b8;

    --muse-border: rgba(15, 23, 42, 0.08);
    --muse-border-strong: rgba(15, 23, 42, 0.14);

    --glass-bg: rgba(255, 255, 255, 0.72);
    --glass-shadow: 0 1px 2px rgba(15, 23, 42, 0.04),
      0 8px 24px rgba(15, 23, 42, 0.06);
    --glass-shadow-hover: 0 2px 4px rgba(15, 23, 42, 0.05),
      0 12px 32px rgba(15, 23, 42, 0.09);

    --radius-card: 18px;
    --radius-control: 12px;

    --font-system: "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI",
      "PingFang SC", "Hiragino Sans GB", "Microsoft YaHei", Arial, sans-serif;
  }

  /* ============================================
   全局背景 — 冰川灰 + 点阵纹理
   ============================================ */
  .chatbox-bg {
    position: relative;
    font-family: var(--font-system);
    color: var(--muse-text);
    background-color: var(--muse-bg);
    background-image: radial-gradient(ellipse 70% 55% at 15% 0%,
        rgba(0, 102, 204, 0.06) 0%,
        transparent 55%),
      radial-gradient(ellipse 55% 45% at 100% 100%,
        rgba(0, 102, 204, 0.04) 0%,
        transparent 55%),
      radial-gradient(circle at 1px 1px,
        rgba(148, 163, 184, 0.22) 1px,
        transparent 0);
    background-size: auto, auto, 26px 26px;
    overflow: hidden;
  }

  .chatbox-bg::before {
    content: "";
    position: absolute;
    inset: 0;
    background-image: linear-gradient(to right,
        rgba(0, 102, 204, 0.03) 1px,
        transparent 1px),
      linear-gradient(to bottom, rgba(0, 102, 204, 0.03) 1px, transparent 1px);
    background-size: 84px 84px;
    pointer-events: none;
    z-index: 0;
  }

  /* ============================================
   导航栏
   ============================================ */
  .chatbox-navbar {
    position: relative;
    z-index: 10;
    padding: 14px 20px 12px;
  }

  .navbar-inner {
    max-width: 880px;
    margin: 0 auto;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding-bottom: 12px;
    border-bottom: 1px solid transparent;
    background-image: linear-gradient(90deg,
        transparent 0%,
        rgba(0, 102, 204, 0.22) 30%,
        rgba(0, 102, 204, 0.22) 70%,
        transparent 100%);
    background-repeat: no-repeat;
    background-position: bottom;
    background-size: 100% 1px;
  }

  .navbar-left {
    display: flex;
    align-items: center;
    gap: 12px;
    min-width: 0;
  }

  .navbar-logo {
    width: 38px;
    height: 38px;
    border-radius: 11px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #fff;
    box-shadow: 0 2px 6px rgba(0, 102, 204, 0.32),
      inset 0 1px 0 rgba(255, 255, 255, 0.2);
    flex-shrink: 0;
  }

  .navbar-logo-text {
    font-size: 17px;
    font-weight: 700;
    line-height: 1;
  }

  .navbar-title {
    display: flex;
    flex-direction: column;
    min-width: 0;
  }

  .navbar-title-text {
    font-size: 16px;
    font-weight: 600;
    color: var(--muse-text);
    letter-spacing: 0.01em;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .navbar-introduce {
    font-size: 12px;
    color: var(--muse-text-secondary);
    margin-top: 2px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .navbar-actions {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .ghost-btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    background: #fff;
    border: 1px solid var(--muse-border-strong);
    color: var(--muse-text-secondary);
    border-radius: 10px;
    font-size: 13px;
    font-weight: 500;
    font-family: var(--font-system);
    padding: 8px 14px;
    cursor: pointer;
    transition: all 0.18s ease;
  }

  .ghost-btn:hover {
    border-color: var(--muse-primary);
    color: var(--muse-primary);
    transform: translateY(-1px);
    box-shadow: 0 3px 10px rgba(0, 102, 204, 0.14);
  }

  /* ============================================
   消息区
   ============================================ */
  .chatbox-main {
    position: relative;
    z-index: 1;
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 12px 20px 20px;
    scroll-behavior: smooth;
  }

  .chatbox-main::-webkit-scrollbar {
    width: 6px;
  }

  .chatbox-main::-webkit-scrollbar-track {
    background: transparent;
  }

  .chatbox-main::-webkit-scrollbar-thumb {
    background: rgba(15, 23, 42, 0.14);
    border-radius: 6px;
  }

  .chatbox-main::-webkit-scrollbar-thumb:hover {
    background: var(--muse-primary);
  }

  /* 空状态 */
  .chatbox-empty {
    max-width: 640px;
    margin: 0 auto;
    padding: 60px 0 40px;
    display: flex;
    flex-direction: column;
    align-items: center;
    text-align: center;
  }

  .empty-rune {
    position: relative;
    width: 96px;
    height: 96px;
    display: flex;
    align-items: center;
    justify-content: center;
    margin-bottom: 20px;
  }

  .empty-ring {
    position: absolute;
    inset: 0;
    border-radius: 50%;
    border: 1.5px solid transparent;
    border-top-color: var(--muse-primary);
    border-right-color: rgba(0, 102, 204, 0.2);
    animation: spin 6s linear infinite;
    box-shadow: 0 0 18px rgba(0, 102, 204, 0.12);
  }

  .empty-icon {
    font-size: 30px;
    color: var(--muse-primary);
    filter: drop-shadow(0 0 10px rgba(0, 102, 204, 0.35));
  }

  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  .empty-title {
    font-size: 22px;
    font-weight: 600;
    color: var(--muse-text);
    margin: 0 0 8px;
    letter-spacing: -0.01em;
  }

  .empty-subtitle {
    font-size: 14px;
    color: var(--muse-text-secondary);
    margin: 0 0 24px;
  }

  .empty-chips {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    justify-content: center;
  }

  .empty-chip {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 14px;
    background: rgba(255, 255, 255, 0.7);
    border: 1px solid var(--muse-border);
    color: var(--muse-text-secondary);
    border-radius: 999px;
    font-size: 13px;
    font-weight: 500;
    font-family: var(--font-system);
    cursor: pointer;
    backdrop-filter: blur(8px);
    transition: all 0.18s ease;
  }

  .empty-chip:hover {
    background: #fff;
    border-color: var(--muse-primary);
    color: var(--muse-primary);
    box-shadow: 0 2px 8px rgba(0, 102, 204, 0.14);
    transform: translateY(-1px);
  }

  .chip-icon {
    filter: grayscale(40%);
    transition: filter 0.2s ease;
  }

  .empty-chip:hover .chip-icon {
    filter: grayscale(0%);
  }

  /* 消息列表 */
  .message-list {
    max-width: 880px;
    margin: 0 auto;
    display: flex;
    flex-direction: column;
    gap: 18px;
  }

  .message-row {
    display: flex;
    gap: 12px;
    align-items: flex-start;
  }

  .message-row--user {
    flex-direction: row-reverse;
  }

  .avatar {
    width: 34px;
    height: 34px;
    border-radius: 10px;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
    font-size: 15px;
  }

  .avatar--assistant {
    background: var(--muse-primary);
    color: #fff;
    font-weight: 700;
    box-shadow: 0 2px 6px rgba(0, 102, 204, 0.28);
  }

  .avatar--user {
    background: #fff;
    color: var(--muse-text-secondary);
    border: 1px solid var(--muse-border);
    font-size: 18px;
  }

  .message-body {
    max-width: calc(100% - 60px);
    display: flex;
    flex-direction: column;
  }

  .message-row--user .message-body {
    align-items: flex-end;
  }

  .message-meta {
    display: flex;
    align-items: baseline;
    gap: 8px;
    margin-bottom: 5px;
    padding: 0 2px;
  }

  .message-row--user .message-meta {
    flex-direction: row-reverse;
  }

  .message-name {
    font-size: 12px;
    font-weight: 600;
    color: var(--muse-text-secondary);
  }

  .message-time {
    font-size: 11px;
    color: var(--muse-text-tertiary);
  }

  .bubble {
    position: relative;
    padding: 12px 16px;
    font-size: 14px;
    line-height: 1.7;
    word-break: break-word;
  }

  .bubble--assistant {
    background: var(--glass-bg);
    border: 1px solid var(--muse-border);
    border-radius: 4px var(--radius-card) var(--radius-card) var(--radius-card);
    box-shadow: var(--glass-shadow);
    backdrop-filter: saturate(160%) blur(14px);
    -webkit-backdrop-filter: saturate(160%) blur(14px);
    color: var(--muse-text);
  }

  .bubble--user {
    background: var(--muse-primary);
    color: #fff;
    border-radius: var(--radius-card) 4px var(--radius-card) var(--radius-card);
    box-shadow: 0 2px 8px rgba(0, 102, 204, 0.28);
  }

  .bubble-content {
    overflow-wrap: anywhere;
  }

  /* 流式光标 */
  .stream-cursor {
    display: inline-block;
    width: 2px;
    height: 15px;
    margin-left: 2px;
    vertical-align: text-bottom;
    background: var(--muse-primary);
    animation: blink 0.9s steps(2) infinite;
  }

  @keyframes blink {
    50% {
      opacity: 0;
    }
  }

  /* 打字点 */
  .typing-dots {
    display: inline-flex;
    gap: 5px;
    padding: 4px 2px;
  }

  .typing-dots i {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: var(--muse-primary);
    animation: bounce 1.2s ease-in-out infinite;
    opacity: 0.4;
  }

  .typing-dots i:nth-child(2) {
    animation-delay: 0.18s;
  }

  .typing-dots i:nth-child(3) {
    animation-delay: 0.36s;
  }

  @keyframes bounce {

    0%,
    80%,
    100% {
      transform: translateY(0);
      opacity: 0.35;
    }

    40% {
      transform: translateY(-7px);
      opacity: 1;
    }
  }

  /* ============================================
   消息操作 — 可配置动作区
   （按钮 / 下拉菜单 / 组合输入框 / 分隔线）
   ============================================ */
  .message-actions {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-top: 5px;
    padding: 0 2px;
    opacity: 0;
    transition: opacity 0.18s ease;
  }

  .message-row:hover .message-actions,
  .message-row:focus-within .message-actions {
    opacity: 1;
  }

  @media (hover: none) {
    .message-actions {
      opacity: 1;
    }
  }

  .mini-btn {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    background: transparent;
    border: none;
    color: var(--muse-text-tertiary);
    font-size: 12px;
    vertical-align: middle;
    font-family: var(--font-system);
    cursor: pointer;
    padding: 3px 8px;
    border-radius: 6px;
    line-height: 1;
    transition: all 0.15s ease;
  }

  .mini-btn:hover:not(:disabled) {
    color: var(--muse-primary);
    background: var(--muse-primary-soft);
  }

  .mini-btn:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }

  .mini-btn--active {
    color: var(--muse-primary);
    background: var(--muse-primary-soft);
  }

  .mini-btn--combo {
    border-left: 1px solid var(--muse-border);
    border-radius: 0 6px 6px 0;
    background: var(--muse-primary-soft);
    color: var(--muse-primary);
    font-weight: 500;
    white-space: nowrap;
  }

  .mini-btn--combo:hover:not(:disabled) {
    background: var(--muse-primary);
    color: #fff;
  }

  /* 分隔线 */
  .action-divider {
    width: 1px;
    height: 14px;
    margin: auto 2px;
    background: var(--muse-border-strong);
    flex: 0 0 auto;
  }

  /* 下拉菜单 */
  .action-dropdown {
    position: relative;
    display: inline-flex;
    align-items: center;
  }

  .action-arrow {
    font-size: 10px;
    opacity: 0.6;
    transition: transform 0.15s ease;
  }

  .action-dropdown.open .action-arrow {
    transform: rotate(180deg);
  }

  .action-menu {
    position: absolute;
    left: 0;
    top: calc(100% + 4px);
    z-index: 30;
    display: flex;
    flex-direction: column;
    gap: 2px;
    min-width: 150px;
    max-height: 260px;
    overflow-y: auto;
    padding: 5px;
    background: #fff;
    border: 1px solid var(--muse-border-strong);
    border-radius: 10px;
    box-shadow: 0 8px 24px rgba(15, 23, 42, 0.14);
  }

  .action-menu-item {
    display: flex;
    align-items: center;
    gap: 7px;
    width: 100%;
    padding: 7px 10px;
    border: none;
    border-radius: 7px;
    background: transparent;
    color: var(--muse-text-secondary);
    font-size: 12.5px;
    font-family: var(--font-system);
    text-align: left;
    white-space: nowrap;
    cursor: pointer;
    transition: background 0.15s ease, color 0.15s ease;
  }

  .action-menu-item:hover:not(:disabled) {
    background: var(--muse-primary-soft);
    color: var(--muse-primary);
  }

  .action-menu-item:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }

  .action-menu-label {
    overflow: hidden;
    text-overflow: ellipsis;
  }

  /* 组合输入框 + 按钮 */
  .action-input-group {
    display: inline-flex;
    align-items: stretch;
    height: 26px;
    background: #fff;
    border: 1px solid var(--muse-border-strong);
    border-radius: 7px;
    overflow: hidden;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;
  }

  .action-input-group:focus-within {
    border-color: var(--muse-primary);
    box-shadow: 0 0 0 2px var(--muse-primary-ring);
  }

  .action-input {
    width: 110px;
    min-width: 60px;
    height: 100%;
    padding: 0 8px;
    border: none;
    outline: none;
    background: transparent;
    color: var(--muse-text);
    font-size: 12px;
    font-family: var(--font-system);
  }

  .action-input::placeholder {
    color: var(--muse-text-tertiary);
  }

  .action-input:disabled {
    opacity: 0.45;
  }

  /* 菜单过渡（兼容 Vue 2 / Vue 3 过渡类名） */
  .action-menu-enter-active,
  .action-menu-leave-active {
    transition: opacity 0.15s ease, transform 0.15s ease;
  }

  .action-menu-enter,
  .action-menu-enter-from,
  .action-menu-leave-to {
    opacity: 0;
    transform: translateY(-4px);
  }

  /* 回到底部 */
  .scroll-bottom-btn {
    position: sticky;
    bottom: 14px;
    left: 50%;
    transform: translateX(-50%);
    display: block;
    margin: 0 auto;
    width: 38px;
    height: 38px;
    border-radius: 50%;
    border: 1px solid var(--muse-border-strong);
    background: #fff;
    color: var(--muse-text-secondary);
    font-size: 16px;
    cursor: pointer;
    box-shadow: 0 4px 14px rgba(15, 23, 42, 0.12);
    transition: all 0.18s ease;
  }

  .scroll-bottom-btn:hover {
    color: var(--muse-primary);
    border-color: var(--muse-primary);
    transform: translateX(-50%) translateY(-1px);
  }

  .fade-up-enter-active,
  .fade-up-leave-active {
    transition: all 0.25s ease;
  }

  .fade-up-enter-from,
  .fade-up-leave-to {
    opacity: 0;
    transform: translateX(-50%) translateY(8px);
  }

  /* ============================================
   Markdown 渲染样式
   ============================================ */
  .bubble-content p {
    margin: 0 0 8px;
  }

  .bubble-content p:last-child {
    margin-bottom: 0;
  }

  .bubble-content h1,
  .bubble-content h2,
  .bubble-content h3,
  .bubble-content h4 {
    font-size: 15px;
    font-weight: 700;
    margin: 14px 0 6px;
    color: var(--muse-text);
  }

  .bubble-content ul,
  .bubble-content ol {
    margin: 6px 0;
    padding-left: 20px;
  }

  .bubble-content li {
    margin: 3px 0;
  }

  .bubble-content code.md-inline {
    background: rgba(0, 102, 204, 0.09);
    color: var(--muse-primary-active);
    border-radius: 5px;
    padding: 1px 6px;
    font-size: 13px;
    font-family: "JetBrains Mono", "Fira Code", Menlo, Consolas, monospace;
  }

  .bubble-content pre.md-code {
    background: #0f172a;
    color: #e2e8f0;
    border-radius: 10px;
    padding: 14px 16px;
    overflow-x: auto;
    margin: 10px 0;
    font-size: 13px;
    line-height: 1.6;
  }

  .bubble-content pre.md-code code {
    font-family: "JetBrains Mono", "Fira Code", Menlo, Consolas, monospace;
    background: transparent;
    color: inherit;
  }

  .bubble-content blockquote {
    border-left: 3px solid var(--muse-primary);
    margin: 8px 0;
    padding: 2px 0 2px 12px;
    color: var(--muse-text-secondary);
  }

  .bubble-content a {
    color: var(--muse-primary);
    text-decoration: none;
    border-bottom: 1px solid rgba(0, 102, 204, 0.3);
  }

  .bubble-content a:hover {
    color: var(--muse-primary-hover);
  }

  .bubble--user .bubble-content code.md-inline {
    background: rgba(255, 255, 255, 0.18);
    color: #fff;
  }

  .bubble--user .bubble-content pre.md-code {
    background: rgba(0, 0, 0, 0.25);
  }

  /* ============================================
   输入区
   ============================================ */
  .chatbox-composer {
    position: relative;
    z-index: 10;
    padding: 10px 20px 18px;
  }

  .composer-inner {
    position: relative;
    max-width: 880px;
    margin: 0 auto;
    background: var(--glass-bg);
    border: 1px solid var(--muse-border);
    border-radius: var(--radius-card);
    box-shadow: var(--glass-shadow);
    backdrop-filter: saturate(160%) blur(16px);
    -webkit-backdrop-filter: saturate(160%) blur(16px);
    padding: 12px 14px 10px;
    transition: box-shadow 0.25s ease, border-color 0.25s ease;
  }

  .composer-inner:focus-within {
    border-color: var(--muse-primary);
    box-shadow: 0 0 0 3px var(--muse-primary-ring), var(--glass-shadow-hover);
  }

  .composer-input {
    width: 100%;
    border: none;
    outline: none;
    resize: none;
    background: transparent;
    font-family: var(--font-system);
    font-size: 14px;
    line-height: 1.6;
    color: var(--muse-text);
    max-height: 180px;
    padding: 2px 2px 6px;
  }

  .composer-input::placeholder {
    color: var(--muse-text-tertiary);
  }

  .composer-footer {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-top: 2px;
  }

  .composer-hint {
    display: flex;
    align-items: center;
    gap: 8px;
    min-height: 26px;
    min-width: 0;
  }

  .model-hint {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 12px;
    color: var(--muse-text-tertiary);
    letter-spacing: 0.01em;
  }

  .streaming-hint {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    font-size: 12px;
    color: var(--muse-primary);
    font-weight: 500;
  }

  .hint-dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: var(--muse-primary);
    animation: pulse 1.2s ease-in-out infinite;
  }

  @keyframes pulse {

    0%,
    100% {
      opacity: 0.35;
      transform: scale(0.9);
    }

    50% {
      opacity: 1;
      transform: scale(1.1);
    }
  }

  .composer-actions {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .send-btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    background: var(--muse-primary);
    border: 1px solid var(--muse-primary);
    color: #fff;
    border-radius: 10px;
    font-size: 13.5px;
    font-weight: 600;
    font-family: var(--font-system);
    padding: 8px 18px;
    cursor: pointer;
    box-shadow: 0 2px 6px rgba(0, 102, 204, 0.32),
      inset 0 1px 0 rgba(255, 255, 255, 0.2);
    transition: all 0.18s cubic-bezier(0.34, 1.2, 0.64, 1);
  }

  .send-btn:hover:not(:disabled) {
    background: var(--muse-primary-hover);
    border-color: var(--muse-primary-hover);
    transform: translateY(-1px);
    box-shadow: 0 6px 16px rgba(0, 102, 204, 0.36);
  }

  .send-btn:active:not(:disabled) {
    transform: translateY(0) scale(0.98);
    background: var(--muse-primary-active);
  }

  .message-body p {
    margin: 0;
  }

  .send-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
    box-shadow: none;
  }

  .send-btn--stop {
    background: #fff;
    border-color: var(--muse-border-strong);
    color: var(--muse-text-secondary);
    box-shadow: 0 1px 3px rgba(15, 23, 42, 0.05);
  }

  .send-btn--stop:hover {
    background: #fff;
    border-color: #dc2626;
    color: #dc2626;
    box-shadow: 0 3px 10px rgba(220, 38, 38, 0.16);
  }

  /* ============================================
   子窗口层 — 同源内嵌 / 最小化 / 索引导航 / 可收起
   ============================================ */
  .child-trigger-btn {
    position: absolute;
    top: -40px;
    right: 6px;
    z-index: 55;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 10px 6px 12px;
    border: 1px solid var(--muse-border-strong);
    border-radius: 999px;
    background: var(--glass-bg);
    backdrop-filter: saturate(160%) blur(16px);
    -webkit-backdrop-filter: saturate(160%) blur(16px);
    box-shadow: var(--glass-shadow);
    color: var(--muse-text-secondary);
    font-family: var(--font-system);
    font-size: 12.5px;
    line-height: 1;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .child-trigger-btn:hover {
    color: var(--muse-primary);
    border-color: rgba(0, 102, 204, 0.35);
    box-shadow: var(--glass-shadow-hover);
    transform: translateY(-1px);
  }

  .child-trigger-btn--hidden {
    opacity: 0.8;
    border-style: dashed;
  }

  .child-trigger-count {
    min-width: 18px;
    height: 18px;
    padding: 0 5px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border-radius: 9px;
    background: var(--muse-primary);
    color: #fff;
    font-size: 11px;
    font-weight: 700;
  }

  .child-window-layer {
    position: absolute;
    right: 20px;
    bottom: 168px;
    width: min(840px, 64vw);
    height: min(62vh, 620px);
    min-height: 240px;
    z-index: 50;
    overflow: hidden;
    background: var(--glass-bg);
    border: 1px solid var(--muse-border-strong);
    border-radius: var(--radius-card);
    box-shadow: 0 16px 48px rgba(15, 23, 42, 0.2), var(--glass-shadow-hover);
    backdrop-filter: saturate(160%) blur(18px);
    -webkit-backdrop-filter: saturate(160%) blur(18px);
  }

  .child-window-nav {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    height: 42px;
    box-sizing: border-box;
    z-index: 2;
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 0 10px;
    border-bottom: 1px solid var(--muse-border);
    background: rgba(255, 255, 255, 0.72);
    box-shadow: 0 1px 2px rgba(15, 23, 42, 0.04);
  }

  .child-nav-count {
    flex: 0 0 auto;
    height: 22px;
    box-sizing: border-box;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 0 9px;
    border-radius: 999px;
    background: var(--muse-primary-soft);
    color: var(--muse-primary-active);
    font-size: 12px;
    font-weight: 600;
  }

  .child-nav-count i {
    font-size: 13px;
  }

  .child-nav-tabs {
    flex: 1 1 auto;
    min-width: 0;
    height: 100%;
    display: flex;
    align-items: center;
    gap: 6px;
    overflow-x: auto;
    scrollbar-width: none;
  }

  .child-nav-tabs::-webkit-scrollbar {
    display: none;
  }

  .child-nav-tab {
    flex: 0 0 auto;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    max-width: 180px;
    height: 28px;
    box-sizing: border-box;
    padding: 0 8px 0 6px;
    border: 1px solid transparent;
    border-radius: 9px;
    background: transparent;
    color: var(--muse-text-secondary);
    font-family: var(--font-system);
    font-size: 12.5px;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .child-nav-tab:hover {
    background: var(--muse-primary-soft);
    color: var(--muse-text);
  }

  .child-nav-tab.active {
    background: var(--muse-primary-soft);
    border-color: rgba(0, 102, 204, 0.28);
    color: var(--muse-primary-active);
    font-weight: 600;
  }

  .child-nav-tab.minimized {
    opacity: 0.55;
  }

  .child-nav-index {
    flex: 0 0 auto;
    width: 18px;
    height: 18px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    background: rgba(0, 102, 204, 0.1);
    color: var(--muse-primary);
    font-size: 11px;
    font-weight: 700;
  }

  .child-nav-tab.active .child-nav-index {
    background: var(--muse-primary);
    color: #fff;
  }

  .child-nav-title {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .child-nav-mini {
    flex: 0 0 auto;
    color: var(--muse-text-tertiary);
    font-size: 12px;
  }

  .child-nav-close {
    flex: 0 0 auto;
    width: 18px;
    height: 18px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    color: var(--muse-text-tertiary);
    font-size: 13px;
    line-height: 1;
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .child-nav-close:hover {
    background: rgba(220, 38, 38, 0.12);
    color: #dc2626;
  }

  .child-nav-actions {
    flex: 0 0 auto;
    height: 100%;
    display: flex;
    align-items: center;
    gap: 4px;
  }

  .child-nav-btn {
    width: 26px;
    height: 26px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border: none;
    border-radius: 7px;
    background: transparent;
    color: var(--muse-text-secondary);
    font-size: 13px;
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .child-nav-btn:hover:not(:disabled) {
    background: rgba(15, 23, 42, 0.07);
    color: var(--muse-text);
  }

  .child-nav-btn:disabled {
    opacity: 0.35;
    cursor: not-allowed;
  }

  .child-nav-btn--danger:hover:not(:disabled) {
    background: rgba(220, 38, 38, 0.1);
    color: #dc2626;
  }

  .child-window-body {
    position: absolute;
    top: 42px;
    left: 0;
    right: 0;
    bottom: 0;
    overflow: hidden;
    background: #fff;
  }

  .child-window-frame {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    border: none;
    background: #fff;
  }

  .child-window-collapsed {
    position: absolute;
    top: 42px;
    left: 0;
    right: 0;
    bottom: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 10px;
    color: var(--muse-text-tertiary);
    font-size: 13px;
  }

  .child-collapsed-icon {
    font-size: 30px;
    opacity: 0.5;
  }

  .child-win-enter-active,
  .child-win-leave-active {
    transition: all 0.28s cubic-bezier(0.34, 1.2, 0.64, 1);
  }

  .child-win-enter-from,
  .child-win-leave-to {
    opacity: 0;
    transform: translateY(14px) scale(0.97);
  }

  /* ============================================
   知识库选择器 — 左上角查看 / 切换（名称 + ID）
   ============================================ */
  .kb-bar {
    position: relative;
    z-index: 11;
    padding: 12px 20px 0;
  }

  .kb-bar-inner {
    max-width: 880px;
    margin: 0 auto;
    display: flex;
    align-items: flex-start;
  }

  .kb-selector {
    position: relative;
    display: inline-flex;
  }

  .kb-trigger {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 6px 12px 6px 8px;
    background: var(--glass-bg);
    border: 1px solid var(--muse-border);
    border-radius: 12px;
    cursor: pointer;
    color: var(--muse-text-secondary);
    font-family: var(--font-system);
    box-shadow: var(--glass-shadow);
    backdrop-filter: saturate(160%) blur(14px);
    -webkit-backdrop-filter: saturate(160%) blur(14px);
    transition: all 0.18s ease;
  }

  .kb-trigger:hover {
    border-color: var(--muse-primary);
    color: var(--muse-text);
    box-shadow: var(--glass-shadow-hover);
    transform: translateY(-1px);
  }

  .kb-trigger-icon {
    flex: 0 0 auto;
    width: 26px;
    height: 26px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border-radius: 8px;
    background: var(--muse-primary-soft);
    color: var(--muse-primary);
    font-size: 14px;
  }

  .kb-trigger-text {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    min-width: 0;
  }

  .kb-trigger-label {
    font-size: 12px;
    color: var(--muse-text-tertiary);
    letter-spacing: 0.02em;
    white-space: nowrap;
  }

  .kb-trigger-name {
    font-size: 13px;
    font-weight: 600;
    color: var(--muse-text);
    max-width: 180px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .kb-trigger-id {
    font-family: "JetBrains Mono", "Fira Code", Menlo, Consolas, monospace;
    font-size: 11px;
    color: var(--muse-text-tertiary);
    background: rgba(15, 23, 42, 0.05);
    border-radius: 6px;
    padding: 1px 6px;
    white-space: nowrap;
  }

  .kb-trigger-empty {
    font-size: 13px;
    color: var(--muse-text-tertiary);
  }

  .kb-trigger-arrow {
    font-size: 11px;
    opacity: 0.6;
    transition: transform 0.18s ease;
  }

  .kb-trigger-arrow.open {
    transform: rotate(180deg);
  }

  .kb-menu {
    position: absolute;
    top: calc(100% + 8px);
    left: 0;
    z-index: 60;
    width: 320px;
    max-width: calc(100vw - 32px);
    background: #fff;
    border: 1px solid var(--muse-border-strong);
    border-radius: 14px;
    box-shadow: 0 16px 48px rgba(15, 23, 42, 0.18);
    overflow: hidden;
    transform-origin: top left;
  }

  .kb-menu-header {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 10px;
    border-bottom: 1px solid var(--muse-border);
  }

  .kb-menu-search {
    flex: 1;
    min-width: 0;
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 7px 10px;
    background: var(--muse-bg);
    border: 1px solid transparent;
    border-radius: 9px;
    color: var(--muse-text-tertiary);
    transition: all 0.15s ease;
  }

  .kb-menu-search:focus-within {
    background: #fff;
    border-color: var(--muse-primary);
    box-shadow: 0 0 0 3px var(--muse-primary-ring);
    color: var(--muse-primary);
  }

  .kb-menu-search input {
    flex: 1;
    min-width: 0;
    border: none;
    outline: none;
    background: transparent;
    font-family: var(--font-system);
    font-size: 13px;
    color: var(--muse-text);
  }

  .kb-menu-search input::placeholder {
    color: var(--muse-text-tertiary);
  }

  .kb-refresh {
    flex: 0 0 auto;
    width: 32px;
    height: 32px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border: none;
    border-radius: 9px;
    background: transparent;
    color: var(--muse-text-secondary);
    font-size: 15px;
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .kb-refresh:hover {
    background: var(--muse-primary-soft);
    color: var(--muse-primary);
  }

  .kb-refresh .spinning {
    animation: spin 0.8s linear infinite;
  }

  .kb-menu-body {
    max-height: 300px;
    overflow-y: auto;
    padding: 6px;
  }

  .kb-menu-body::-webkit-scrollbar {
    width: 6px;
  }

  .kb-menu-body::-webkit-scrollbar-thumb {
    background: rgba(15, 23, 42, 0.14);
    border-radius: 6px;
  }

  .kb-menu-status {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    padding: 28px 12px;
    color: var(--muse-text-tertiary);
    font-size: 13px;
  }

  .kb-menu-status i {
    font-size: 22px;
    opacity: 0.6;
  }

  .kb-spinner {
    width: 18px;
    height: 18px;
    border: 2px solid rgba(0, 102, 204, 0.2);
    border-top-color: var(--muse-primary);
    border-radius: 50%;
    animation: spin 0.8s linear infinite;
  }

  .kb-item {
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 10px;
    width: 100%;
    padding: 9px 10px;
    border: none;
    border-radius: 10px;
    background: transparent;
    text-align: left;
    cursor: pointer;
    transition: background 0.15s ease;
  }

  .kb-item:hover {
    background: var(--muse-primary-soft);
  }

  .kb-item.active {
    background: var(--muse-primary-soft);
    box-shadow: inset 0 0 0 1px rgba(0, 102, 204, 0.18);
  }

  .kb-item-icon {
    flex: 0 0 auto;
    width: 28px;
    height: 28px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border-radius: 8px;
    background: rgba(0, 102, 204, 0.08);
    color: var(--muse-primary);
    font-size: 14px;
    margin-top: 1px;
  }

  .kb-item-info {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  .kb-item-name {
    font-size: 13px;
    font-weight: 600;
    color: var(--muse-text);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .kb-item.active .kb-item-name {
    color: var(--muse-primary-active);
  }

  .kb-item-id {
    font-family: "JetBrains Mono", "Fira Code", Menlo, Consolas, monospace;
    font-size: 11px;
    color: var(--muse-text-tertiary);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .kb-item-check {
    flex: 0 0 auto;
    color: var(--muse-primary);
    font-size: 15px;
    margin-top: 6px;
  }

  .kb-item-copy {
    flex: 0 0 auto;
    color: var(--muse-text-tertiary);
    font-size: 13px;
    margin-top: 4px;
    padding: 3px;
    border-radius: 6px;
    opacity: 0;
    transition: opacity 0.15s ease, color 0.15s ease;
  }

  .kb-item:hover .kb-item-copy {
    opacity: 1;
  }

  .kb-item-copy:hover {
    color: var(--muse-primary);
  }

  .kb-menu-enter-active,
  .kb-menu-leave-active {
    transition: opacity 0.18s ease, transform 0.18s ease;
  }

  .kb-menu-enter,
  .kb-menu-enter-from,
  .kb-menu-leave-to {
    opacity: 0;
    transform: translateY(-6px) scale(0.98);
  }

  /* 输入区下方的知识库提示胶囊 */
  .kb-hint {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    max-width: 180px;
    padding: 2px 9px;
    border: none;
    border-radius: 999px;
    background: rgba(0, 102, 204, 0.06);
    color: var(--muse-text-tertiary);
    font-size: 11px;
    font-family: var(--font-system);
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .kb-hint i {
    flex: 0 0 auto;
    font-size: 11px;
  }

  .kb-hint span {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .kb-hint:hover {
    color: var(--muse-primary);
    background: var(--muse-primary-soft);
  }

  .kb-menu-divider {
    height: 1px;
    margin: 4px 6px;
    background: var(--muse-border);
  }

  .kb-item--auto .kb-item-icon {
    background: rgba(139, 92, 246, 0.1);
    color: #8b5cf6;
  }

  /* ============================================
   响应式
   ============================================ */
  @media (max-width: 640px) {
    .chatbox-navbar {
      padding: 10px 12px 8px;
    }

    .chatbox-main {
      padding: 10px 12px 14px;
    }

    .chatbox-composer {
      padding: 8px 12px 12px;
    }

    .navbar-introduce {
      display: none;
    }

    .message-body {
      max-width: calc(100% - 50px);
    }

    .child-window-layer {
      left: 12px;
      right: 12px;
      bottom: 120px;
      width: auto;
      height: min(68vh, 600px);
    }

    .child-nav-title {
      max-width: 96px;
    }

    .child-trigger-btn {
      top: -36px;
      right: 8px;
      padding: 5px 9px 5px 10px;
      font-size: 12px;
    }

    .child-trigger-text {
      display: none;
    }

    .kb-bar {
      padding: 8px 12px 0;
    }

    .kb-trigger-name {
      max-width: 110px;
    }

    .kb-menu {
      width: 280px;
    }
  }
</style>

<script>export default 
          {"name":"ChatBox","data":function(
) {
return {
      loading: false,
      loadingText: "正在思考…",
      inputPlaceholder: "输入你的问题，Enter 发送，Shift+Enter 换行",
      isStreaming: false,
      abortController: null,
      messageActions: [{
            type: "button",
            key: "copy",
            label: "复制",
            icon: "bi bi-copy",
          },
          {
            type: "button",
            key: "skill",
            id: "4099f7a380a5898232e2e9e2c2459cd0",
            label: "生成 PPT",
            icon: "bi bi-copy",
          },
          {
            type: "button",
            key: "skill",
            id: "1f8dc55b3d900b5313650399a55afe03",
            label: "生成图表",
            icon: "bi bi-copy",
          },
          {
            type: "divider",
          },
          {
            type: "dropdown",
            key: "more",
            label: "更多",
            children: [{
              key: "regen",
              label: "重新生成",
              icon: "bi bi-arrow-repeat",
            }, ],
          },
          {
            type: "input-button",
            key: "skill",
            placeholder: "输入 AI Skill ID",
            buttonText: ">",
          },
        ],
      draft: "",
      openMenuId: "",
      showScrollHint: false,
      messages: [],
      title: "ChatBox",
      introduce: "与 AI 对话功能。",
      documentMeta: webCpu.documentMeta,
      knowledgeBase: "",
      icon: "聊",
      exampleList: [{
            icon: "💡",
            name: "帮我构思一个面向年轻人的低成本周末户外活动方案",
          },
          {
            icon: "✍️",
            name: "写一段小红书风格的夏季防晒霜种草文案，带 Emoji",
          },
          {
            icon: "🧮",
            name: "用通俗易懂的大白话和生活中的比喻，向新手解释什么是量子计算",
          },
          {
            icon: "📋",
            name: "提炼这篇 2000 字会议纪要的核心结论，并列出后续待办事项（Action Items）",
          },
        ],
      query: {"apiName":"深度求索","model":"deepseek-v4-flash","documentId":"0f2b0ef4bc23dd23ccc3e588af72854e"},
      content: {
          inputText: "",
          outputText: "",
        },
      accountData: {},
      isSolo: WebTool.urlQuery(location.href, "solo"),
      dataEditable: true,
      defaultTemplate: "-",
      // 子窗口管理：当前窗口内打开的同源子页面（iframe），支持最小化与索引导航
        childWindows: [],
      activeChildId: "",
      // 子窗口浮窗面板显示/隐藏（隐藏后仅保留输入框右上方的控制按钮）
        childPanelVisible: true,
      // 知识库选择：左上角查看 / 切换（名称 + ID 标识）
        knowledgeBaseId: "",
      kbLoading: false,
      kbSearch: "",
      kbMenuOpen: false,
      // 知识库决策接口请求中（避免并发触发）；空知识库（knowledgeBaseId 为空）即自动决策模式
        kbAnalyzing: false
    };
},"computed":{"canSend":function() {
        return !!(this.draft && this.draft.trim()) && !this.isStreaming;
      },"documentId":function() {
        let documentId = this.query.documentId;
        if (
          !documentId &&
          webCpu.currentCard &&
          webCpu.currentCard.task &&
          webCpu.currentCard.task.vueItem &&
          webCpu.currentCard.task.vueItem.documentMeta
        ) {
          documentId = webCpu.currentCard.task.vueItem.documentMeta.documentId;
        }
        return documentId;
      },"effectiveMessageActions":function() {
        // 动作区配置优先级：props.messageActions > content.messageActions > 默认复制按钮
        if (Array.isArray(this.messageActions) && this.messageActions.length) {
          return this.messageActions;
        }
        if (
          this.content &&
          Array.isArray(this.content.messageActions) &&
          this.content.messageActions.length
        ) {
          return this.content.messageActions;
        }
        return this.defaultMessageActions();
      },"activeChildWindow":function() {
        return this.childWindows.find((w) => w.id === this.activeChildId) || null;
      },"currentKnowledgeBase":function() {
        // 空知识库（knowledgeBaseId 为空）表示自动决策模式，返回 null
        if (!this.knowledgeBaseId || !this.knowledgeBases.length) {
          return null;
        }
        return (
          this.knowledgeBases.find((kb) => kb.id === this.knowledgeBaseId) || null
        );
      },"filteredKnowledgeBases":function() {
        const q = (this.kbSearch || "").trim().toLowerCase();
        if (!q) {
          return this.knowledgeBases;
        }
        return this.knowledgeBases.filter((kb) => {
          const name = String(kb.name || "").toLowerCase();
          const id = String(kb.id || "").toLowerCase();
          const desc = String(kb.description || "").toLowerCase();
          return (
            name.indexOf(q) !== -1 ||
            id.indexOf(q) !== -1 ||
            desc.indexOf(q) !== -1
          );
        });
      },"kbStorageKey":function() {
        return "kb_selected_" + (this.documentId || "default");
      }},"methods":{"actionHandler":function(value) {
        // 消息动作接管处理：
        // - skill：不再新开浏览器标签页，而是在当前窗口内打开同源子页面（inner iframe），
        //   并纳入子窗口管理器：支持最小化；打开多个时仅展示当前激活窗口，
        //   其余自动最小化，顶部“索引导航”可随时切换 / 恢复 / 关闭。
        let item = (value && value.item) || value || {};
        let message = (value && value.message) || value || {};

        // 携带初始输入给子页面（同源 localStorage 传递）
        if (message && message.content) {
          localStorage.agentInput = message.content;
        }

        if (item.key === "skill") {
          const url = this.buildChildWindowUrl(item, value);
          if (url) {
            this.openChildWindow({
              url: url,
              title: item.label || item.name || "子窗口",
            });
            // 返回 false 表示已接管该动作，跳过默认行为
            return false;
          }
        }
        return undefined;
      },"buildChildWindowUrl":function(item, value) {
        // 优先取按钮配置的 id，其次取组合输入框提交的 Skill ID（value 可能为字符串）
        const id =
          (item && item.id) ||
          (value && typeof value === "object" && value.value) ||
          (typeof value === "string" && value) ||
          "";
        if (!id) {
          return "";
        }
        try {
          // 以当前地址为基座，仅替换查询参数，保持同源开子页
          const url = new URL(location.href.split("#")[0]);
          url.searchParams.set("id", String(id));
          url.searchParams.set("solo", "true");
          url.searchParams.set("auto", "true");
          return url.href;
        } catch (e) {
          return `${location.origin}?id=${encodeURIComponent(
          id
        )}&solo=true&auto=true`;
        }
      },"openChildWindow":function(opts) {
        opts = opts || {};
        const win = {
          id: "cw_" +
            Date.now().toString(36) +
            Math.random().toString(36).slice(2, 8),
          title: opts.title || "子窗口",
          url: opts.url || "",
          minimized: false,
          createdAt: Date.now(),
        };
        // 保证同时只显示一个子窗口：新窗口激活，其余全部最小化
        this.childWindows.forEach((w) => {
          w.minimized = true;
        });
        this.childWindows.push(win);
        this.activeChildId = win.id;
        // 新开任务时自动展开面板，确保用户能看到新窗口
        this.childPanelVisible = true;
        return win;
      },"toggleChildPanel":function() {
        if (!this.childWindows.length) {
          return;
        }
        this.childPanelVisible = !this.childPanelVisible;
      },"activateChildWindow":function(id) {
        const win = this.childWindows.find((w) => w.id === id);
        if (!win) {
          return;
        }
        // 激活目标窗口，其余自动最小化
        this.childWindows.forEach((w) => {
          w.minimized = w.id !== id;
        });
        this.activeChildId = id;
      },"minimizeChildWindow":function(id) {
        const win = this.childWindows.find((w) => w.id === id);
        if (!win || win.minimized) {
          return;
        }
        win.minimized = true;
        if (this.activeChildId === id) {
          // 当前激活窗口被最小化后，自动激活最近一个未最小化窗口；没有则全部收起
          const next = this.childWindows
            .slice()
            .reverse()
            .find((w) => !w.minimized);
          this.activeChildId = next ? next.id : "";
        }
      },"closeChildWindow":function(id) {
        this.$confirm(
          "关闭后无法重新打开，确认要关闭该窗口吗？",
          webCpu.messages[lang].confirmDialogTitle, {
            confirmButtonText: webCpu.messages[lang].confirmYes,
            cancelButtonText: webCpu.messages[lang].confirmNo,
            type: "warning",
          }
        ).then(() => {
          const idx = this.childWindows.findIndex((w) => w.id === id);
          if (idx === -1) {
            return;
          }
          this.childWindows.splice(idx, 1);
          if (this.activeChildId === id) {
            // 关闭激活窗口后，自动激活最近一个窗口，否则清空激活状态
            const next = this.childWindows
              .slice()
              .reverse()
              .find((w) => !w.minimized);
            const fallback = this.childWindows[this.childWindows.length - 1];
            const target = next || fallback;
            if (target) {
              target.minimized = false;
              this.activeChildId = target.id;
            } else {
              this.activeChildId = "";
            }
          }
        });
      },"send":async function() {
        const text = (this.draft || "").trim();
        if (!text || this.isStreaming) {
          return false;
        }

        this.pushMessage("user", text);
        this.draft = "";
        this.autoResize();
        this.showScrollHint = false;

        // 助手占位
        const assistant = this.pushMessage("assistant", "", true);

        this.startStream(assistant.id);
      },"startStream":function(assistantId) {
        this.isStreaming = true;
        this.loading = true;

        const query = Object.assign({}, this.query);
        query.documentId = this.documentId;

        query.context = this.buildContext();

        this.abortController = aigcInterface.streamRequest(query, {
          immediateFinish: false,
          onMessage: (accumulated) => {
            // 实时收到内容：逐字更新到消息气泡
            this.updateMessage(assistantId, {
              content: accumulated || "",
            });
            this.scrollToBottom();
          },
          onStreamFinished: (result) => {
            this.updateMessage(assistantId, {
              content: result || "",
              streaming: false,
            });
            this.content.outputText = result || "";
            this.finishStream();
          },
          onError: () => {
            const msg = this.getMessage(assistantId);
            if (msg && !msg.content) {
              this.updateMessage(assistantId, {
                content: "请求出现异常，请稍后重试。",
              });
            }
            this.updateMessage(assistantId, {
              streaming: false,
            });
            this.finishStream();
          },
          onAction: (action) => {
            this.updateMessage(assistantId, {
              streaming: false,
            });
            this.finishStream();
            if (action === "LOGIN") {
              aigcInterface.openLoginDialog(document.body, () => {
                this.retryStream(assistantId);
              });
            } else if (action === "RECHARGE") {
              aigcInterface.renderRechargeDialog(document.body);
            }
          },
        });
      },"retryStream":function(assistantId) {
        this.updateMessage(assistantId, {
          content: "",
          streaming: true,
        });
        this.startStream(assistantId);
      },"regenerate":function() {
        // 正在输出时先停止当前流，再重新生成
        if (this.isStreaming) {
          this.stopStreaming();
        }
        const assistant = this.lastAssistant();
        if (!assistant) {
          return false;
        }
        const idx = this.messages.findIndex((m) => m.id === assistant.id);
        if (idx === -1) {
          return false;
        }
        // 移除最后一条助手回复（连同未完成的半截内容），
        // 上下文里仍保留触发它的用户提问，随后对新占位消息重新发起请求
        this.messages.splice(idx, 1);
        this.content.outputText = "";
        const newAssistant = this.pushMessage("assistant", "", true);
        this.showScrollHint = false;
        this.startStream(newAssistant.id);
        return true;
      },"finishStream":function() {
        this.isStreaming = false;
        this.loading = false;
        this.abortController = null;
        this.scrollToBottom();
      },"stopStreaming":function() {
        if (
          this.abortController &&
          typeof this.abortController.abort === "function"
        ) {
          this.abortController.abort();
        }
        const assistant = this.lastAssistant();
        if (assistant) {
          this.updateMessage(assistant.id, {
            streaming: false,
          });
        }
        this.finishStream();
      },"pushMessage":function(role, content, streaming) {
        const msg = {
          id: Date.now().toString(36) + Math.random().toString(36).slice(2, 7),
          role: role,
          content: content || "",
          time: Date.now(),
          streaming: !!streaming,
        };
        this.messages.push(msg);
        this.$nextTick(() => this.scrollToBottom());
        // 返回响应式代理，确保后续流式更新能触发渲染
        return this.messages[this.messages.length - 1];
      },"getMessage":function(id) {
        return this.messages.find((m) => m.id === id) || null;
      },"updateMessage":function(id, patch) {
        const idx = this.messages.findIndex((m) => m.id === id);
        if (idx !== -1) {
          Object.assign(this.messages[idx], patch);
        }
      },"lastAssistant":function() {
        for (let i = this.messages.length - 1; i >= 0; i--) {
          if (this.messages[i].role === "assistant") {
            return this.messages[i];
          }
        }
        return null;
      },"buildContext":function() {
        const history = this.messages
          .filter((m) => m.content && String(m.content).trim())
          .slice(-24);
        return history
          .map((m) => {
            const who = m.role === "user" ? "用户" : "助手";
            return `${who}：${m.content}`;
          })
          .join("\n");
      },"clearChat":function() {
        if (this.isStreaming) {
          this.stopStreaming();
        }
        this.messages = [];
        this.content.outputText = "";
        this.scrollToBottom(true);
      },"useExample":function(d) {
        this.draft = d.name || d.title || "";
        this.$nextTick(() => {
          this.autoResize();
          if (this.$refs.textarea) {
            this.$refs.textarea.focus();
          }
        });
      },"defaultMessageActions":function() {
        return [{
          type: "button",
          key: "copy",
          label: "复制",
          icon: "bi bi-copy",
          tooltip: "复制到剪贴板",
        }, ];
      },"isComboItem":function(item) {
        if (!item || typeof item !== "object") {
          return false;
        }
        return (
          item.type === "input-button" ||
          item.type === "inputButton" ||
          item.type === "input" ||
          item.type === "combo"
        );
      },"menuKey":function(m, item) {
        return m.id + "::" + item.key;
      },"isMenuOpen":function(m, item) {
        return this.openMenuId === this.menuKey(m, item);
      },"toggleActionMenu":function(m, item) {
        const key = this.menuKey(m, item);
        const wasOpen = this.openMenuId === key;
        this.openMenuId = wasOpen ? "" : key;
        this.dispatchMessageAction({
          type: wasOpen ? "close" : "open",
          key: item.key,
          item: item,
          message: m,
        });
      },"selectActionChild":function(m, parent, child) {
        this.openMenuId = "";
        this.dispatchMessageAction({
          type: "select",
          key: child.key,
          parentKey: parent.key,
          value: child.value !== undefined ? child.value : undefined,
          item: child,
          parent: parent,
          message: m,
        });
      },"onActionInput":function(m, item, event) {
        item.value = event.target.value;
      },"submitCombo":function(m, item, event) {
        if (item.disabled) {
          return;
        }
        const group =
          event && event.target && event.target.closest ?
          event.target.closest(".action-input-group") :
          null;
        const input = group ? group.querySelector("input") : null;
        this.dispatchMessageAction({
          type: "submit",
          key: item.key,
          value: input ? input.value : "",
          item: item,
          message: m,
        });
      },"dispatchMessageAction":function(action) {
        // 外部自定义处理：返回 false 表示已接管，跳过默认行为
        const handled = this.actionHandler ?
          this.actionHandler(action) :
          undefined;
        if (handled === false) {
          return;
        }
        this.$emit("action", action);
        this.applyDefaultAction(action);
      },"applyDefaultAction":function(action) {
        // 命中“点击 / 下拉选择”的内置默认动作
        const isDefaultAction = (keys) =>
          (action.type === "click" || action.type === "select") &&
          keys.indexOf(action.key) !== -1;
        // copy → 复制消息内容
        if (isDefaultAction(["copy"])) {
          this.copyText(action.message && action.message.content);
        }
        // regen / regenerate / 重新生成 → 重新生成最后一条助手回复
        else if (isDefaultAction(["regen", "regenerate", "重新生成"])) {
          this.regenerate();
        }
      },"copyText":function(text) {
        if (WebTool && WebTool.copyString) {
          WebTool.copyString(text);
          this.$message({
            type: "success",
            message: "已复制到剪贴板。",
          });
        }
      },"formatTime":function(ts) {
        const d = new Date(ts || Date.now());
        const pad = (n) => (n < 10 ? "0" + n : n);
        return `${pad(d.getHours())}:${pad(d.getMinutes())}`;
      },"onKeydown":function(e) {
        if (
          e.key === "Enter" &&
          !e.shiftKey &&
          !e.isComposing &&
          !e.altKey &&
          !e.ctrlKey &&
          !e.metaKey
        ) {
          e.preventDefault();
          this.send();
        }
      },"autoResize":function() {
        this.$nextTick(() => {
          const el = this.$refs.textarea;
          if (!el) {
            return;
          }
          el.style.height = "auto";
          el.style.height = Math.min(el.scrollHeight, 180) + "px";
        });
      },"onScroll":function() {
        const el = this.$refs.scrollContainer;
        if (!el) {
          return;
        }
        const distance = el.scrollHeight - el.scrollTop - el.clientHeight;
        this.showScrollHint = distance > 120;
      },"scrollToBottom":function(smooth) {
        this.$nextTick(() => {
          const el = this.$refs.scrollContainer;
          if (!el) {
            return;
          }
          el.scrollTo({
            top: el.scrollHeight,
            behavior: smooth ? "smooth" : "auto",
          });
        });
      },"renderMarkdown":function(text) {
        if (!text) {
          return "";
        }
        const escapeHtml = (s) =>
          String(s)
          .replace(/&/g, "&amp;")
          .replace(/</g, "&lt;")
          .replace(/>/g, "&gt;");

        const inline = (s) =>
          escapeHtml(s)
          .replace(/`([^`\n]+)`/g, '<code class="md-inline">$1</code>')
          .replace(/\*\*([^*]+)\*\*/g, "<strong>$1</strong>")
          .replace(
            /\[([^\]]+)\]\((https?:\/\/[^\s)]+)\)/g,
            '<a href="$2" target="_blank" rel="noopener noreferrer">$1</a>'
          );

        const lines = String(text).split("\n");
        const out = [];
        let codeBuf = null;
        let listType = null;

        const flushList = () => {
          if (listType) {
            out.push(`</${listType}>`);
            listType = null;
          }
        };

        for (let raw of lines) {
          const line = raw.replace(/\r$/, "");

          // 代码块
          if (/^```/.test(line.trim())) {
            if (codeBuf === null) {
              flushList();
              codeBuf = [];
            } else {
              out.push(
                `<pre class="md-code"><code>${escapeHtml(
                codeBuf.join("\n")
              )}</code></pre>`
              );
              codeBuf = null;
            }
            continue;
          }
          if (codeBuf !== null) {
            codeBuf.push(line);
            continue;
          }

          const t = line.trim();
          if (!t) {
            flushList();
            continue;
          }

          // 标题
          let m = t.match(/^(#{1,4})\s+(.*)$/);
          if (m) {
            flushList();
            out.push(`<h4>${inline(m[2])}</h4>`);
            continue;
          }

          // 引用
          if (t.startsWith(">")) {
            flushList();
            out.push(
              `<blockquote>${inline(t.replace(/^>\s?/, ""))}</blockquote>`
            );
            continue;
          }

          // 无序列表
          m = t.match(/^[-*+]\s+(.*)$/);
          if (m) {
            if (listType !== "ul") {
              flushList();
              out.push('<ul class="md-list">');
              listType = "ul";
            }
            out.push(`<li>${inline(m[1])}</li>`);
            continue;
          }

          // 有序列表
          m = t.match(/^\d+[.)]\s+(.*)$/);
          if (m) {
            if (listType !== "ol") {
              flushList();
              out.push('<ol class="md-list">');
              listType = "ol";
            }
            out.push(`<li>${inline(m[1])}</li>`);
            continue;
          }

          // 普通段落
          flushList();
          out.push(`<p>${inline(t)}</p>`);
        }

        if (codeBuf !== null) {
          out.push(
            `<pre class="md-code"><code>${escapeHtml(
            codeBuf.join("\n")
          )}</code></pre>`
          );
        }
        flushList();

        return out.join("");
      }},"mounted":async function() {
      this.content = this.content || {};



      // Esc 关闭各类弹层
      document.addEventListener("keydown", this.onGlobalKeydown);


      // 平台传入的初始指令
      if (aigcInterface && typeof aigcInterface.handleAgentInput === "function") {
        aigcInterface.handleAgentInput(location.href, (str) => {
          if (str) {
            this.draft = str;
            this.autoResize();
            const auto = WebTool.urlQuery(location.href, "auto");
            if (auto) {
              this.send();
            }
          }
        });
      }
    },"beforeDestroy":function() {
      document.removeEventListener("keydown", this.onGlobalKeydown);
    },"appMap":{"loginRegItem":{"url":"/mainApp/app/loginRegItem.js","key":"transweb_loginRegItem","cardName":"loginRegCard","dsl":{"data":{"position":"top","loginedMenu":[{"label":"⏱️ 历史记录","action":function(d, callback) {
                dialogMap.openDocumentApp(
                  webCpu.appConfig.aigcHistory, {
                    data: {
                      content: {
                        isSolo: true,
                      },
                      query: {
                        documentId: webCpu.currentVueItem.query.documentId,
                      },
                    },
                  }, {
                    title: "⏱️ 历史记录",
                    callback: callback,
                  }, {
                    width: "100%",
                    height: "100%",
                  }
                );
              }}]}}}}};</script>