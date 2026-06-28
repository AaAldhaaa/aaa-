// public/scripts/extensions/third-party/GodComment/index.js
import {
    eventSource,
    eventTypes,
    getContext,
    saveSettingsDebounced,
} from '../../../../script.js';

// 确保全局扩展设置对象存在
window.extension_settings = window.extension_settings || {};

// 默认设置
const defaultSettings = {
    enabled: true,
    api_base: 'https://api.openai.com',
    api_key: '',
    model: 'gpt-3.5-turbo',
    system_prompt:
        '你是一个上帝视角的评论者。请用一两句话，以诙谐、犀利或温暖的旁观者角度，对下面的角色发言进行评论。直接输出评论内容，不要添加前缀或引号。',
};

// 正在请求中的消息 ID 集合，防止重复调用
const pendingRequests = new Set();

// 初始化/加载设置
function loadSettings() {
    if (!extension_settings.god_comment) {
        extension_settings.god_comment = structuredClone(defaultSettings);
    }
    // 补全可能缺失的字段
    const current = extension_settings.god_comment;
    for (const key of Object.keys(defaultSettings)) {
        if (current[key] === undefined) {
            current[key] = defaultSettings[key];
        }
    }
}

// 保存设置
function saveSettings() {
    saveSettingsDebounced();
}

// 调用 OpenAI 兼容接口获取评论
async function fetchGodComment(messageText) {
    const settings = extension_settings.god_comment;
    if (!settings.api_key || !settings.api_base) {
        throw new Error('API Key 或 Base URL 未配置');
    }

    const url = `${settings.api_base.replace(/\/+$/, '')}/v1/chat/completions`;
    const headers = {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${settings.api_key}`,
    };
    const body = JSON.stringify({
        model: settings.model,
        messages: [
            { role: 'system', content: settings.system_prompt },
            { role: 'user', content: messageText },
        ],
        temperature: 0.7,
        max_tokens: 120,
    });

    const response = await fetch(url, {
        method: 'POST',
        headers,
        body,
    });

    if (!response.ok) {
        const errorText = await response.text();
        throw new Error(`API 请求失败 (${response.status}): ${errorText}`);
    }

    const data = await response.json();
    const comment = data.choices?.[0]?.message?.content?.trim();
    if (!comment) {
        throw new Error('API 返回了空的评论内容');
    }
    return comment;
}

// 为指定消息 ID 的 DOM 元素添加或更新评论 UI
function renderCommentInDOM(msgId, commentText, isLoading = false) {
    const messageElement = document.getElementById(`chat-msg-${msgId}`);
    if (!messageElement) return;

    let commentDiv = messageElement.querySelector('.god-comment');
    if (!commentDiv) {
        commentDiv = document.createElement('div');
        commentDiv.className = 'god-comment';
        messageElement.appendChild(commentDiv);
    }

    if (isLoading) {
        commentDiv.innerHTML = '<small><em>🕊️ 上帝视角评论：</em> 思考中...</small>';
    } else {
        commentDiv.innerHTML = `<small><em>🕊️ 上帝视角评论：</em> ${escapeHtml(commentText)}</small>`;
    }
}

// 简单的 HTML 转义，避免 XSS
function escapeHtml(text) {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}

// 当角色消息渲染完毕时触发
async function onCharacterMessageRendered(msgId) {
    if (!extension_settings.god_comment.enabled) return;

    const context = getContext();
    const msg = context.chat[msgId];
    if (!msg || !msg.is_char) return;   // 只处理角色消息

    // 如果消息对象里已经有评论，直接显示
    if (msg.extra?.god_comment) {
        renderCommentInDOM(msgId, msg.extra.god_comment);
        return;
    }

    // 如果正在请求中，不再重复触发
    if (pendingRequests.has(msgId)) return;

    // 没有评论且未请求过，开始获取
    pendingRequests.add(msgId);
    renderCommentInDOM(msgId, null, true); // 显示加载状态

    try {
        const comment = await fetchGodComment(msg.mes);
        // 保存到消息的 extra 字段中
        if (!msg.extra) msg.extra = {};
        msg.extra.god_comment = comment;
        // 更新 UI
        renderCommentInDOM(msgId, comment);
        // 保存聊天文件
        await context.saveChatDebounced();
    } catch (error) {
        console.error('上帝视角评论获取失败:', error);
        renderCommentInDOM(msgId, '获取评论失败，请检查 API 配置或网络。');
    } finally {
        pendingRequests.delete(msgId);
    }
}

// 消息被删除时清除 extra 中的评论（可选）
function onMessageDeleted(msgId) {
    const msg = getContext().chat[msgId];
    if (msg?.extra) {
        delete msg.extra.god_comment;
    }
}

// 消息更新（如编辑、重新生成）时，清除旧评论，下次渲染会重新生成
function onMessageUpdated(msgId) {
    const msg = getContext().chat[msgId];
    if (msg?.extra) {
        delete msg.extra.god_comment;
    }
    // 移除 DOM 中的评论元素
    const msgEl = document.getElementById(`chat-msg-${msgId}`);
    if (msgEl) {
        const commentDiv = msgEl.querySelector('.god-comment');
        if (commentDiv) commentDiv.remove();
    }
}

// 注册事件
function registerEvents() {
    eventSource.on(eventTypes.CHARACTER_MESSAGE_RENDERED, onCharacterMessageRendered);
    eventSource.on(eventTypes.MESSAGE_DELETED, onMessageDeleted);
    eventSource.on(eventTypes.MESSAGE_UPDATED, onMessageUpdated);
}

// 生成扩展设置面板 HTML，并绑定事件
function addSettingsPanel() {
    const settingsHtml = `
    <div id="god_comment_settings" class="extension_settings">
      <h4>上帝视角评论</h4>
      <div class="inline-drawer">
        <label class="checkbox_label">
          <input type="checkbox" id="god_comment_enabled" />
          <span>启用自动评论</span>
        </label>
        <div class="inline-drawer-content">
          <label>API Base URL<br/>
            <input type="text" id="god_comment_api_base" class="text_pole" placeholder="https://api.openai.com" />
          </label>
          <label>API Key<br/>
            <input type="password" id="god_comment_api_key" class="text_pole" placeholder="sk-..." />
          </label>
          <label>模型<br/>
            <input type="text" id="god_comment_model" class="text_pole" placeholder="gpt-3.5-turbo" />
          </label>
          <label>系统提示词<br/>
            <textarea id="god_comment_system_prompt" class="text_pole" rows="4" placeholder="你是一个上帝视角的评论者..."></textarea>
          </label>
        </div>
      </div>
    </div>`;

    // 将面板插入扩展设置区域
    $('#extensions_settings').append(settingsHtml);

    // 绑定输入事件
    const s = extension_settings.god_comment;
    $('#god_comment_enabled')
        .prop('checked', s.enabled)
        .on('change', function () {
            s.enabled = $(this).prop('checked');
            saveSettings();
        });
    $('#god_comment_api_base')
        .val(s.api_base)
        .on('input', function () {
            s.api_base = $(this).val();
            saveSettings();
        });
    $('#god_comment_api_key')
        .val(s.api_key)
        .on('input', function () {
            s.api_key = $(this).val();
            saveSettings();
        });
    $('#god_comment_model')
        .val(s.model)
        .on('input', function () {
            s.model = $(this).val();
            saveSettings();
        });
    $('#god_comment_system_prompt')
        .val(s.system_prompt)
        .on('input', function () {
            s.system_prompt = $(this).val();
            saveSettings();
        });
}

// 启动
jQuery(() => {
    loadSettings();
    registerEvents();
    addSettingsPanel();
});
