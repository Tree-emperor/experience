#!/usr/bin/env node
/**
 * spec-sps-subagent-driven-development 状态机脚本 (TypeScript 版本)
 *
 * 基于大模型决策的状态机：
 * 1. 读取 todoList.json 和 scheduleLog.md
 * 2. 检查工作报告是否已更新
 * 3. 调用 Claude/CodeAgent 进行决策
 * 4. 输出 JSON 结果给主Agent
 *
 * SDK 切换：
 * - USE_CODEAGENT_SDK=true  使用 CodeAgent SDK
 * - 默认                   使用 Claude SDK
 */

import * as fs from 'fs/promises';
import * as fsSync from 'fs';
import * as path from 'path';
import * as crypto from 'crypto';
import { spawn } from 'child_process';

// ==================== SDK 切换配置 ====================
const USE_CODEAGENT_SDK = process.env.USE_CODEAGENT_SDK === 'true';

// ==================== Claude SDK 类型定义 ====================

interface ClaudeMessage {
  type?: string;
  result?: string;
}

interface ClaudeOptions {
  permissionMode?: string;
  [key: string]: unknown;
}

interface ClaudeQueryResult {
  [Symbol.asyncIterator](): AsyncIterator<ClaudeMessage>;
}

// SDK 函数类型定义
type StartupFunction = (config?: { options?: ClaudeOptions }) => Promise<{ query: QueryFunction }>;
type QueryFunction = (prompt: string) => ClaudeQueryResult;

// CodeAgent SDK 类型定义
interface CodeAgentClient {
  provider: {
    list: () => Promise<{ data: { all: Array<{ id: string; models: Record<string, unknown> }> } }>;
  };
  session: {
    create: (params: Record<string, unknown>) => Promise<{ data?: { id: string }; error?: string }>;
    prompt: (params: {
      sessionID: string;
      model: { providerID: string; modelID: string };
      parts: Array<{ type: string; text: string }>;
    }) => Promise<{ data?: { parts: Array<{ type: string; text: string }> }; info?: { id: string } }>;
  };
  event: {
    subscribe: () => Promise<{ stream: AsyncIterable<unknown> }>;
  };
  permission: {
    reply: (params: { requestID: string; reply: string }) => Promise<unknown>;
  };
}

type CreateOpencodeClientFunction = (config: {
  baseUrl: string;
  throwOnError: boolean;
  directory: string;
}) => CodeAgentClient;

// 动态导入 SDK
let startup: StartupFunction | null = null;
let warmInstance: { query: QueryFunction } | null = null;

// CodeAgent SDK 变量
let codeagentClient: CodeAgentClient | null = null;
const CODEAGENT_BASE_URL = process.env.CODEAGENT_BASE_URL || 'http://127.0.0.1:31547';
const CODEAGENT_PORT = parseInt(process.env.CODEAGENT_PORT || '31547', 10);

// ==================== CodeAgent Server 启动函数 ====================

/**
 * 检查端口是否被占用
 */
function isPortInUse(port: number): Promise<boolean> {
  return new Promise((resolve) => {
    const net = require('net');
    const client = new net.Socket();
    client.once('connect', () => {
      client.destroy();
      resolve(true);
    });
    client.once('error', () => {
      client.destroy();
      resolve(false);
    });
    client.connect(port, '127.0.0.1');
    // 超时保护
    setTimeout(() => {
      client.destroy();
      resolve(false);
    }, 1000);
  });
}

/**
 * 启动 nga serve (隐藏窗口)
 */
function startNgaServe(): Promise<void> {
  return new Promise((resolve, reject) => {
    debugLog('[启动] 正在启动 nga serve...');

    // 构建启动命令
    const serveArgs = `serve --disable-update --port ${CODEAGENT_PORT}`;

    // 创建临时的 VBScript 脚本文件来隐藏窗口启动
    const vbsContent = `Set WshShell = CreateObject("WScript.Shell")\nWshShell.Run "nga ${serveArgs}", 0, False`;
    const vbsPath = path.join(process.env.TEMP || 'C:\\Temp', `nga_serve_${Date.now()}.vbs`);

    try {
      fsSync.writeFileSync(vbsPath, vbsContent, 'utf8');
    } catch (e) {
      reject(new Error(`无法创建临时脚本文件: ${e}`));
      return;
    }

    const cmdStr = `cscript.exe //Nologo "${vbsPath}"`;
    debugLog('[启动] 执行命令:', cmdStr);

    const child = spawn('cscript.exe', ['//Nologo', vbsPath], {
      stdio: DEBUG_MODE ? ['ignore', 'pipe', 'pipe'] : 'ignore',
      detached: true,
      windowsHide: true,
    });

    // 仅在 debug 模式下打印 child 日志
    if (DEBUG_MODE) {
      console.error(`[nga-serve command] ${cmdStr}`);
      console.error(`[nga-serve script] ${vbsContent}`);

      child.stdout?.on('data', (data: Buffer) => {
        const lines = data.toString().trim().split('\n');
        for (const line of lines) {
          if (line.trim()) {
            console.error(`[nga-serve stdout] ${line}`);
          }
        }
      });

      child.stderr?.on('data', (data: Buffer) => {
        const lines = data.toString().trim().split('\n');
        for (const line of lines) {
          if (line.trim()) {
            console.error(`[nga-serve stderr] ${line}`);
          }
        }
      });
    }

    child.unref();

    // 延迟删除临时脚本文件（给进程一点时间启动）
    setTimeout(() => {
      try {
        fsSync.unlinkSync(vbsPath);
      } catch {
        // 忽略删除错误
      }
    }, 2000);

    // 等待进程启动后返回
    setTimeout(() => {
      debugLog('[启动] nga serve 进程已启动');
      resolve();
    }, 1000);
  });
}

/**
 * 确保 CodeAgent Server 启动成功
 */
async function ensureCodeAgentServer(): Promise<void> {
  // 检查端口是否已有服务运行
  const inUse = await isPortInUse(CODEAGENT_PORT);
  if (inUse) {
    debugLog(`[启动] nga serve 已在端口 ${CODEAGENT_PORT} 运行，跳过启动`);
    return;
  }

  await startNgaServe();

  // 等待服务启动，最多等待 30 秒
  const maxWaitTime = 30000;
  const checkInterval = 1000;
  let waitedTime = 0;

  debugLog(`[启动] 等待 nga serve 启动...`);

  while (waitedTime < maxWaitTime) {
    const serverReady = await isPortInUse(CODEAGENT_PORT);
    if (serverReady) {
      debugLog(`[启动] nga serve 已成功启动，端口 ${CODEAGENT_PORT} 可用`);
      return;
    }
    await new Promise(resolve => setTimeout(resolve, checkInterval));
    waitedTime += checkInterval;
  }

  throw new Error(`nga serve 启动超时（${maxWaitTime / 1000}秒），请检查安装`);
}

// 获取全局 node_modules 路径
function getGlobalNodeModulesPath(): string {
  const { execSync } = require('child_process');
  try {
    return execSync('npm root -g', { encoding: 'utf-8' }).trim();
  } catch {
    return process.env.NODE_PATH || '';
  }
}

const globalNodeModulesPath = getGlobalNodeModulesPath();

// 根据环境变量加载对应 SDK
if (USE_CODEAGENT_SDK) {
  // CodeAgent SDK 懒加载，client 初始化在 callCodeAgentAsync 中完成
  const codeagentSdkPaths = [
    path.join(globalNodeModulesPath, '@codeagent-sdk', 'js', 'dist', 'v2', 'client.js'),
  ];

  let loaded = false;
  for (const sdkPath of codeagentSdkPaths) {
    if (fsSync.existsSync(sdkPath)) {
      loaded = true;
      break;
    }
  }

  if (!loaded) {
    console.error('CodeAgent SDK 未安装，请执行以下命令完成依赖安装：');
    console.error('1. npm config set @codeagent-sdk:registry "https://cmc.centralrepo.rnd.huawei.com/artifactory/api/npm/product_npm/"');
    console.error('2. npm install -g tsx @codeagent-sdk/js@1.2.27-alpha.20260409.2');
    process.exit(1);
  }
} else {
  // 加载 Claude SDK
  const claudeAgentSdkPath = globalNodeModulesPath
    ? path.join(globalNodeModulesPath, '@anthropic-ai', 'claude-agent-sdk')
    : '';

  try {
    const sdk = claudeAgentSdkPath
      ? require(claudeAgentSdkPath)
      : require('@anthropic-ai/claude-agent-sdk');
    startup = sdk.startup as StartupFunction;
  } catch (e) {
    // SDK 未安装，使用空实现
    console.error('@anthropic-ai/claude-agent-sdk 未安装，请使用`npm install -g tsx @anthropic-ai/claude-agent-sdk@0.3.153`完成依赖安装');
    process.exit(1);
  }
}

// 预热 Claude SDK
async function warmupClaude(): Promise<void> {
  if (!startup) {
    throw new Error("Claude SDK 未安装，请运行 npm install -g @anthropic-ai/claude-agent-sdk@0.3.153");
  }
  warmInstance = await startup({ options: { permissionMode: 'bypassPermissions' } });
}

// 全局调试标志
let DEBUG_MODE = false;

// 关键提示 - 用于在长程任务中保持 Agent 的注意力
const CRITICAL_HINT = "1.**连续执行**：在执行`mindspec:spec-exec`和`mindspec:spec-sps-subagent-driven-development`技能的过程中，禁止中途询问用户是否继续，用户未主动打断就一直往下执行。\n 2.**状态机权威**：严格遵守技能`mindspec:spec-sps-subagent-driven-development`的要求，充分信任状态机结果，按状态机的action/message/reason记录scheduleLog.md并派发SubAgent执行，禁止自行判断或跳过步骤。记录scheduleLog.md时，需调用命令获取实际时间。\n3.**禁止自行更新ToDoList。**\n4.**全Plan闭环**：状态机返回finish时，若有剩余plan文件待执行，需继续按`mindspec:spec-sps-subagent-driven-development`执行剩余Plan文件，直至全部完成。";

// 模板文件路径映射
const TEMPLATE_FILES: Record<string, string> = {
  "implementer": "implementer-prompt.md",
  "spec_reviewer": "spec-reviewer-prompt.md",
  "quality_reviewer": "code-quality-reviewer-prompt.md",
  "clean_reviewer": "code-clean-reviewer-prompt.md",
  "security_reviewer": "code-security-reviewer-prompt.md",
};

// Agent 对应的默认工作报告文件名
const AGENT_TO_DEFAULT_FILENAME: Record<string, string> = {
  "implementer": "implementer.md",
  "spec_reviewer": "spec-reviewer.md",
  "quality_reviewer": "code-quality-reviewer.md",
  "clean_reviewer": "code-clean-reviewer.md",
  "security_reviewer": "code-security-reviewer.md",
};

// 决策到模板的映射
const DECISION_TO_TEMPLATE: Record<string, string> = {
  "do-implement": "implementer",
  "do-parallel-review": "implementer",  // 并发审查时需要 implementer 的 prompt
  "do-implement-fix": "implementer",
  "do-spec-review": "spec_reviewer",
  "do-code-quality-review": "quality_reviewer",
  "do-code-clean-review": "clean_reviewer",
  "do-code-security-review": "security_reviewer",
};

// 决策到 action 的映射
const DECISION_TO_ACTION: Record<string, string> = {
  "do-implement": "dispatch_implementer",
  "do-parallel-review": "dispatch_parallel",
  "do-implement-fix": "dispatch_implementer",
  "nextTodo": "nextTodo",
  "finish": "finish",
  "regenerate": "regenerate",
  "update-work-report": "update_work_report",
};

// Agent 类型到决策的映射（用于断点恢复）
const AGENT_TYPE_TO_DECISION: Record<string, string> = {
  "implementer": "do-implement",
  "spec_reviewer": "do-spec-review",
  "quality_reviewer": "do-code-quality-review",
  "clean_reviewer": "do-code-clean-review",
  "security_reviewer": "do-code-security-review",
};

// 决策到审查类型的映射
const DECISION_TO_REVIEW_TYPE: Record<string, string> = {
  "do-spec-review": "spec",
  "do-code-quality-review": "code_quality",
  "do-code-clean-review": "code_clean",
  "do-code-security-review": "code_security",
};

// 审查类型到 action 的映射（用于从 review_type 生成 parallel_tasks）
const REVIEW_TYPE_TO_ACTION: Record<string, string> = {
  "spec": "dispatch_spec_reviewer",
  "code_quality": "dispatch_quality_reviewer",
  "code_clean": "dispatch_clean_reviewer",
  "code_security": "dispatch_security_reviewer",
};

// 审查类型到 decision 的映射（用于从 review_type 生成 parallel_tasks）
const REVIEW_TYPE_TO_DECISION: Record<string, string> = {
  "spec": "do-spec-review",
  "code_quality": "do-code-quality-review",
  "code_clean": "do-code-clean-review",
  "code_security": "do-code-security-review",
};

// ==================== 类型定义 ====================

// 并发任务单元
interface ParallelTask {
  action: string;        // "dispatch_spec_reviewer" 等
  decision: string;      // "do-parallel-review" 等
  prompt: string;        // 生成的 prompt
  report_path: string;   // 目标报告路径
  review_type: string;   // "spec", "code_quality", "code_clean", "code_security"
}

interface TodoItem {
  id: string;
  name: string;
  status: string;
  review_status?: Record<string, string>;
  retry_count?: number;  // 当前已重试次数（implementer 修复 + reviewer 重审为一轮）
  max_retry?: number;    // 最大重试次数，固定为 3
  description?: string;
  context?: string;
  title?: string;
}

interface TodoList {
  todos: TodoItem[];
  current_todo_id?: string;
  current_todo_index?: number;  // 新增：当前 todo 索引
  current_agent?: string | string[];  // 支持数组
  updated_at?: string;
  plan_file?: string;
  working_directory?: string;
  plan_hash?: string;
  session_id?: string;  // 当前 session ID，用于 compact 后判断是否需要提醒
}

interface BreakpointRecoveryResult {
  need_recovery: boolean;
  recovery_agent?: string | string[];
  action: string;
  decision?: string;
  reason?: string;
  message: string;
  parallel_tasks?: ParallelTask[];  // 并发任务列表
  is_breakpoint_recovery?: boolean;
  completed_reviews?: string[];
}

interface DecisionResult {
  action: string;
  decision: string;
  reason: string;
  prompt: string;
  message: string;
  error: string | null;
  critical_hint?: string;
  is_breakpoint_recovery?: boolean;
  parallel_tasks?: ParallelTask[];  // 并发任务列表
  review_status_update?: Record<string, string>;  // 审查状态更新映射
}

// Decision 枚举值类型
type DecisionType = "do-implement" | "do-parallel-review" | "do-implement-fix" | "nextTodo" | "finish";

// CurrentTodoInfo 类型定义
interface CurrentTodoInfo {
  description: string;
  context: string;
  title: string;
  name: string;
}

interface FixWorkingDirectoryResult {
  fixed: boolean;
  old_working_directory: string;
  working_directory: string;
  reason: string;
}

// ==================== 断点恢复辅助函数 ====================

function buildReportPaths(
  todoDir: string,
  reviewStatus?: Record<string, string>,
  currentAgent?: string
): Record<string, string[]> {
  const keyToKeywords: Record<string, string[]> = {
    "implementer": ["implementer"],
    "spec_reviewer": ["spec_reviewer"],
    "quality_reviewer": ["code_quality_reviewer", "quality_reviewer"],
    "clean_reviewer": ["code_clean_reviewer", "clean_reviewer"],
    "security_reviewer": ["code_security_reviewer", "security_reviewer"],
  };

  // 用 searchFilesInDirectory 搜索 todoDir
  const result = searchFilesInDirectory(todoDir, AGENT_TO_DEFAULT_FILENAME, keyToKeywords);

  // ========== 容错：在父级目录搜索所有文件 ==========
  if (reviewStatus && currentAgent) {
    const parentDir = path.dirname(todoDir);

    // 防止根目录无限循环
    if (parentDir === todoDir) {
      debugLog(`[DEBUG] 已在根目录，跳过父级目录搜索`);
    } else {
      // 在父级目录搜索所有 key
      const parentResult = searchFilesInDirectory(parentDir, AGENT_TO_DEFAULT_FILENAME, keyToKeywords);

      // 对每个 key，比较父级目录找到的文件与已有的文件
      for (const key of Object.keys(AGENT_TO_DEFAULT_FILENAME)) {
        const parentFiles = parentResult[key] || [];
        const existingFiles = result[key] || [];

        if (parentFiles.length === 0) continue;

        // 获取已有文件的最新修改时间
        let existingLatestMtime = 0;
        for (const file of existingFiles) {
          try {
            const mtime = fsSync.statSync(file).mtimeMs;
            if (mtime > existingLatestMtime) {
              existingLatestMtime = mtime;
            }
          } catch {
            // 忽略
          }
        }

        // 添加父级目录中找到的、修改时间比已有文件更新的文件
        for (const file of parentFiles) {
          // 检查是否已存在于已有列表中
          if (existingFiles.includes(file)) continue;

          try {
            const mtime = fsSync.statSync(file).mtimeMs;
            if (mtime > existingLatestMtime) {
              result[key].push(file);
              debugLog(`[DEBUG] 父级目录容错添加文件: ${key} -> ${file}`);
            }
          } catch {
            // 忽略
          }
        }
      }
    } // end else (非根目录)
  }
  // ========== 容错结束 ==========

  return result;
}

function escapeRegExp(string: string): string {
  return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

/**
 * 获取当前时间的本地 ISO 格式字符串
 * 用于更新 todoList.updated_at，确保与文件系统时间一致
 */
function toLocalISOString(): string {
  const now = new Date();
  // toLocaleString 返回本地时区的可读时间，再转换为 Date 对象获取标准化格式
  const localStr = now.toLocaleString('sv-SE', { hour12: false }).replace(' ', 'T');
  return localStr;
}

/**
 * 将 Date 对象转换为本地时区的 ISO 格式字符串
 */
function toLocalISOStringFromDate(date: Date): string {
  const localStr = date.toLocaleString('sv-SE', { hour12: false }).replace(' ', 'T');
  return localStr;
}

/**
 * 在指定目录中搜索匹配的文件路径列表
 * @param dir 要搜索的目录
 * @param defaultFilenames 默认文件名映射
 * @param keyToKeywords 关键词映射
 * @param targetKeys 只搜索这些 key，为空则搜索所有
 * @returns 每个 key 对应的匹配文件路径列表
 */
/**
 * 递归获取目录中所有 .md 文件（排除 scheduleLog.md）
 */
function getAllMdFiles(dir: string): string[] {
  const files: string[] = [];

  function traverse(currentDir: string): void {
    try {
      const entries = fsSync.readdirSync(currentDir, { withFileTypes: true });
      for (const entry of entries) {
        const fullPath = path.join(currentDir, entry.name);
        if (entry.isDirectory()) {
          traverse(fullPath);
        } else if (entry.isFile() && entry.name.endsWith('.md') && entry.name !== 'scheduleLog.md') {
          files.push(fullPath);
        }
      }
    } catch {
      // 忽略访问错误
    }
  }

  traverse(dir);
  return files;
}

function searchFilesInDirectory(
  dir: string,
  defaultFilenames: Record<string, string>,
  keyToKeywords: Record<string, string[]>,
  targetKeys?: string[]
): Record<string, string[]> {
  const result: Record<string, string[]> = {};
  const seenPaths: Record<string, Set<string>> = {};
  const keys = targetKeys || Object.keys(defaultFilenames);

  for (const key of keys) {
    result[key] = [];
    seenPaths[key] = new Set();
  }

  try {
    // 递归获取所有 md 文件（排除 scheduleLog.md）
    const files = getAllMdFiles(dir);

    for (const filePath of files) {
      const fileName = path.basename(filePath);
      const fileNameLower = fileName.toLowerCase();

      for (const key of keys) {
        const defaultName = defaultFilenames[key].toLowerCase();
        const filenameMatch = fileNameLower.includes(defaultName);

        let contentMatch = false;
        if (!filenameMatch) {
          const keywords = keyToKeywords[key] || [key];
          try {
            const content = fsSync.readFileSync(filePath, 'utf-8').toLowerCase();
            for (const keyword of keywords) {
              const regex = new RegExp(`\\b${escapeRegExp(keyword)}\\b`, 'i');
              if (regex.test(content)) {
                contentMatch = true;
                break;
              }
            }
          } catch {
            // 忽略读取错误
          }
        }

        if (filenameMatch || contentMatch) {
          if (!seenPaths[key].has(filePath)) {
            // 检查文件名是否是其他 key 的默认文件名，如果是则跳过
            let isOtherKeyDefaultFilename = false;
            for (const otherKey of Object.keys(defaultFilenames)) {
              if (otherKey !== key && fileNameLower.includes(defaultFilenames[otherKey].toLowerCase())) {
                isOtherKeyDefaultFilename = true;
                break;
              }
            }
            if (!isOtherKeyDefaultFilename) {
              result[key].push(filePath);
              seenPaths[key].add(filePath);
            }
          }
        }
      }
    }

    // 排序：默认文件名优先，然后按字母顺序
    for (const key of keys) {
      result[key].sort((a, b) => {
        const aName = path.basename(a).toLowerCase();
        const bName = path.basename(b).toLowerCase();
        const defaultName = defaultFilenames[key].toLowerCase();
        const aScore = aName === defaultName ? 0 : 1;
        const bScore = bName === defaultName ? 0 : 1;
        if (aScore !== bScore) return aScore - bScore;
        return a.localeCompare(b);
      });
    }
  } catch {
    // 目录不存在时返回空结果
  }

  return result;
}

function checkAllTasksCompleted(todoList: TodoList): boolean {
  const todos = todoList.todos || [];
  if (!todos.length) return true;
  for (const todo of todos) {
    // 允许的状态：COMPLETED, finished, finished_with_warnings
    const validStatuses = ["COMPLETED", "finished", "finished_with_warnings"];
    if (!validStatuses.includes(todo.status)) return false;

    // 检查所有 review_status 都是 Passed
    const reviewStatus = todo.review_status || {};
    for (const [key, value] of Object.entries(reviewStatus)) {
      if (value !== "Passed") {
        return false;
      }
    }
  }
  return true;
}

function checkPlanChanged(todoList: TodoList): boolean {
  const storedHash = todoList.plan_hash || "";
  const planFile = todoList.plan_file || "";
  if (!planFile || !storedHash) return false;
  const currentHash = computePlanHash(planFile);
  return currentHash !== storedHash;
}

function computePlanHash(planFile: string): string {
  try {
    const planPath = path.resolve(planFile);
    if (!fsSync.existsSync(planPath)) return "";
    const planContent = fsSync.readFileSync(planPath, 'utf-8');
    return crypto.createHash('md5').update(planContent).digest('hex');
  } catch {
    return "";
  }
}

function extractReportTimestamp(reportPath: string): string {
  // 优先使用文件修改时间（mtime），与 Python 逻辑一致
  try {
    if (fsSync.existsSync(reportPath)) {
      const stats = fsSync.statSync(reportPath);
      const mtime = stats.mtime;
      // 转换为本地时区的 ISO 格式字符串
      return toLocalISOStringFromDate(mtime);
    }
  } catch {
    // 忽略错误，继续从内容匹配
  }

  // 获取不到时，从内容中匹配时间戳
  try {
    const content = fsSync.readFileSync(reportPath, 'utf-8');
    const patterns = [
      /(\d{4}[-/]\d{1,2}[-/]\d{1,2}[T\s]\d{1,2}:\d{1,2}:\d{1,2}(?:\.\d+)?(?:Z|[+-]\d{2}:?\d{2})?)/g,
      /(\d{4}[-/]\d{1,2}[-/]\d{1,2}[T\s]\d{1,2}:\d{1,2}:\d{1,2})/g,
    ];

    const allTimestamps: string[] = [];

    for (const pattern of patterns) {
      const matches = content.matchAll(pattern);
      for (const match of Array.from(matches)) {
        let timestamp = match[1];
        // 统一转换为 ISO 格式
        timestamp = timestamp.replace('/', '-').replace(' ', 'T');
        // 去除时区信息
        timestamp = timestamp.replace(/[+-]\d{2}:?\d{2}$/, '').replace(/Z$/, '');
        allTimestamps.push(timestamp);
      }
    }

    // 返回所有匹配中最大的时间戳
    if (allTimestamps.length > 0) {
      // 使用日期比较获取最大值，处理无效日期
      const validTimestamps = allTimestamps.filter(t => !isNaN(new Date(t).getTime()));
      if (validTimestamps.length === 0) return "";
      const sorted = validTimestamps.sort((a, b) => new Date(b).getTime() - new Date(a).getTime());
      return sorted[0] || "";
    }
  } catch {
    // 忽略错误
  }
  return "";
}

async function checkBreakpointRecovery(
  todoList: TodoList,
  todoDir: string,
  todoListPath: string,
  reportPaths: Record<string, string[]>
): Promise<BreakpointRecoveryResult> {
  // 1. 检查所有任务是否已完成
  if (checkAllTasksCompleted(todoList)) {
    return {
      need_recovery: true,
      recovery_agent: "finish",
      decision: "finish",
      reason: "所有任务已完成，无需继续执行"
    };
  }

  // 2. 检查 Plan 是否发生变化
  if (checkPlanChanged(todoList)) {
    return {
      need_recovery: true,
      recovery_agent: "regenerate",
      decision: "regenerate",
      reason: "plan_hash发生变化，请备份旧文件并根据Plan中的未完成项重新生成todoList.json。如果确认Plan文件未变化，则删除plan_hash字段以继续执行。"
    };
  }

  // 3. 获取断点恢复检测所需信息
  const todoListUpdated = todoList.updated_at || "";

  // 支持 current_agent 为数组
  const currentAgents = Array.isArray(todoList.current_agent)
    ? todoList.current_agent
    : todoList.current_agent ? [todoList.current_agent] : [];

  if (currentAgents.length === 0 || !todoListUpdated) {
    return { need_recovery: false };
  }

  // 获取当前 Todo
  const currentTodoId = todoList.current_todo_id || "";
  const currentTodo = (todoList.todos || []).find(t => t.id === currentTodoId);
  const reviewStatus = currentTodo?.review_status || {};

  // 4. 构建需要恢复的 parallel_tasks（implementer 和 reviewer 统一处理逻辑）
  const pendingTasks: ParallelTask[] = [];
  const completedAgents: string[] = [];

  for (const agentType of currentAgents) {
    const paths = reportPaths[agentType] || [];
    let hasNewReport = false;

    for (const reportPath of paths) {
      const reportTime = extractReportTimestamp(reportPath);
      if (reportTime && reportTime > todoListUpdated) {
        hasNewReport = true;
        break;
      }
    }

    if (hasNewReport) {
      completedAgents.push(agentType);
      continue;
    }

    // 需要恢复 - 确定 decision 和 action
    let decision: string;
    let action: string;

    if (agentType === "implementer") {
      // implementer 的 decision 根据 review_status 判断
      const allPending = Object.values(reviewStatus).every(status => status === "Pending");
      decision = allPending ? "do-implement" : "do-implement-fix";
      action = "dispatch_implementer";
    } else {
      decision = getDecisionByAgentType(agentType);
      const templateKey = DECISION_TO_TEMPLATE[decision] || "";
      action = `dispatch_${templateKey}`;
    }

    // 生成提示词，获取 report_path 和 review_type
    const templateKey = agentType === "implementer" ? "implementer" : DECISION_TO_TEMPLATE[decision] || "";
    const prompt = await generatePrompt(decision, currentTodo!, todoList, todoListPath, reportPaths);
    const reportPath = getReportPathByAgentType(templateKey, todoDir);
    const reviewType = getReviewTypeByAgentType(templateKey);

    pendingTasks.push({
      action,
      decision,
      prompt,
      report_path: reportPath,
      review_type: reviewType
    });
  }

  // 5. 返回恢复结果
  if (pendingTasks.length > 0) {
    const isSingleTask = pendingTasks.length === 1;

    return {
      need_recovery: true,
      decision: isSingleTask ? pendingTasks[0].decision : "do-parallel-review",
      action: isSingleTask ? pendingTasks[0].action : "dispatch_parallel",
      parallel_tasks: pendingTasks,
      reason: `断点恢复：${pendingTasks.map(t => t.action).join(", ")} 执行异常。请参考\`message\`完成恢复处理`,
      message: `**处理步骤**：
1. 首先检查报告是否已生成（可能在其他位置或文件名不同）
2. 如果已生成但不在目标路径 → 移动到目标路径（不覆盖已有文件，冲突时追加数字后缀）
3. 如果确实未生成 → 之前的SubAgent已执行异常，直接按 parallel_tasks 中的 prompt **重新派发** SubAgent来推进任务

**待处理任务**：
${pendingTasks.map(t => `- ${t.action}: ${t.report_path || "(无报告路径)"}`).join("\n")}`,
      is_breakpoint_recovery: true
    };
  }

  // 6. 全部完成，继续正常流程
  return { need_recovery: false };
}

// ==================== 工具函数 ====================

function getTemplateDir(): string {
  // Python 版本使用 Path(__file__).parent.parent
  return path.dirname(path.dirname(__filename));
}

function getDocsDir(): string {
  return path.join(getTemplateDir(), 'docs');
}

function sanitizeFolderName(name: string): string {
  let result = name.replace(/[<>:"/\\|?*]/g, '_');
  result = result.substring(0, 50);
  result = result.replace(/_+/g, '_');
  // TypeScript 的 trim() 不接受参数，使用正则替代
  result = result.replace(/^_+|_+$/g, '') || "todo";
  return result;
}

async function readTemplate(templateKey: string): Promise<string> {
  const templateFile = path.join(getTemplateDir(), TEMPLATE_FILES[templateKey]);
  try {
    return await fs.readFile(templateFile, 'utf-8');
  } catch {
    throw new Error(`模板文件不存在: ${templateFile}`);
  }
}

async function readTodoList(todoListPath: string): Promise<TodoList> {
  const content = await fs.readFile(todoListPath, 'utf-8');
  return JSON.parse(content);
}

async function writeTodoList(todoListPath: string, todoList: TodoList): Promise<void> {
  await fs.writeFile(todoListPath, JSON.stringify(todoList, null, 2), 'utf-8');
}

async function ensureTodoDir(todoDir: string): Promise<void> {
  await fs.mkdir(todoDir, { recursive: true });
}

function debugLog(...messages: unknown[]): void {
  if (DEBUG_MODE) {
    console.error(...messages);
  }
}

// ==================== Prompt 辅助函数 ====================

function extractChangeName(planFile: string, todoListPath: string): string {
  const match = planFile.match(/[\/\\]openspec[\/\\]changes[\/\\]([^/\\]+)[\/\\]plans[\/\\]/);
  if (match) {
    return match[1];
  }
  return path.basename(path.dirname(todoListPath));
}

function getNextTodoInfo(todoList: TodoList, currentTodoId: string): CurrentTodoInfo {
  const todos = todoList.todos || [];
  const currentIndex = todos.findIndex(t => t.id === currentTodoId);
  const nextTodo = todos[currentIndex + 1];
  if (nextTodo) {
    return {
      description: (nextTodo as { description?: string }).description || "",
      context: (nextTodo as { context?: string }).context || "",
      title: (nextTodo as { title?: string }).title || nextTodo.name || "",
      name: nextTodo.name || ""
    };
  }
  return { description: "", context: "", title: "", name: "" };
}

function getCurrentTodoInfo(todoList: TodoList, decision: string, currentTodo: TodoItem): CurrentTodoInfo {
  if (decision === "nextTodo") {
    const nextInfo = getNextTodoInfo(todoList, currentTodo.id);
    if (nextInfo.name) {
      return nextInfo;
    }
  }
  return {
    description: (currentTodo as { description?: string }).description || "",
    context: (currentTodo as { context?: string }).context || "",
    title: (currentTodo as { title?: string }).title || currentTodo.name || "",
    name: currentTodo.name || ""
  };
}

function getImplementerReportsAsList(reportPaths: string[]): string {
  const validPaths = reportPaths.filter(p => fsSync.existsSync(p));
  if (validPaths.length === 0) return "无";
  return validPaths.map(p => `- ${p}`).join("\n");
}

function getLatestReportFileByType(reportType: string, reportPaths: Record<string, string[]>): string {
  const paths = reportPaths[reportType] || [];
  if (paths.length === 0) return "";

  let latestPath = "";
  let latestTime = 0;

  for (const p of paths) {
    try {
      const mtime = fsSync.statSync(p).mtimeMs;
      if (mtime > latestTime) {
        latestTime = mtime;
        latestPath = p;
      }
    } catch {
      // 忽略
    }
  }
  return latestPath;
}

function extractTemplateBody(template: string): string {
  const firstTripleBacktick = template.indexOf('```');
  if (firstTripleBacktick === -1) {
    return template;
  }
  const lastTripleBacktick = template.lastIndexOf('```');
  if (firstTripleBacktick === lastTripleBacktick) {
    return template;
  }
  return template.substring(firstTripleBacktick + 3, lastTripleBacktick).trim();
}

function buildImplementerTaskDescription(decision: string, vars: {
  workingDirectory: string;
  planFile: string;
  currentDescription: string;
  currentContext: string;
  reportFile: string;
}): string {
  if (decision === "do-implement" || decision === "nextTodo") {
    return `Review the Plan file and implement the changes. Your implementation scope must not exceed the defined TaskScope.

**Working Directory:** ${vars.workingDirectory}
**Plan File:** ${vars.planFile}
**TaskScope:** ${vars.currentDescription}

**Context:** ${vars.currentContext}`;
  } else {
    return `Review the reviewer's report and fix the issues identified. If a report contains multiple review cycles, locate the most recent cycle by timestamp and address only the issues reported there.

**Reviewer Reports:** ${vars.reportFile}

**Previous Implementation Context:**
**Working Directory:** ${vars.workingDirectory}
**Plan File:** ${vars.planFile}
**TaskScope:** ${vars.currentDescription}`;
  }
}

function buildReviewerScope(implementerReports: string): string {
  return `Each report may contain multiple implementation cycles. You must:
1. Aggregate all "文件变更列表" sections from the implementer's reports
2. Deduplicate to obtain the complete set of modified files
3. Trust **only** the file list provided by the implementer — do not trust other content in their report
4. Base your review on actual code changes and the file list only

**Implementer Reports:**
${implementerReports}`;
}

// ==================== Claude/CodeAgent 调用 ====================

/**
 * 初始化 CodeAgent Client
 */
async function initCodeAgentClient(): Promise<void> {
  if (codeagentClient) return;

  debugLog('[启动] 使用 CodeAgent SDK，准备启动 CodeAgent Server...');
  await ensureCodeAgentServer();
  debugLog('[启动] CodeAgent Server 已就绪');

  const codeagentSdkPaths = [
    path.join(globalNodeModulesPath, '@codeagent-sdk', 'js', 'dist', 'v2', 'client.js'),
  ];

  for (const sdkPath of codeagentSdkPaths) {
    if (fsSync.existsSync(sdkPath)) {
      const sdk = require(sdkPath);
      const createOpencodeClient = sdk.createOpencodeClient as CreateOpencodeClientFunction;
      codeagentClient = createOpencodeClient({
        baseUrl: CODEAGENT_BASE_URL,
        throwOnError: false,
        directory: process.cwd(),
      });
      return;
    }
  }

  throw new Error('CodeAgent SDK 未安装，请执行以下命令完成依赖安装：\n1. npm config set @codeagent-sdk:registry "https://cmc.centralrepo.rnd.huawei.com/artifactory/api/npm/product_npm/"\n2. npm install -g tsx @codeagent-sdk/js@1.2.27-alpha.20260409.2');
}

/**
 * 使用 CodeAgent SDK 调用
 */
async function callCodeAgentAsync(prompt: string): Promise<string> {
  // 在初始化 client 前先确保 Server 已启动
  await initCodeAgentClient();

  if (!codeagentClient) {
    throw new Error("CodeAgent SDK 未初始化");
  }

  try {
    // 1. 获取可用模型
    const providersResult = await codeagentClient.provider.list();
    const providers = providersResult.data?.all || [];

    if (!providers.length) {
      throw new Error('No provider found');
    }

    const useProviderID = 'w3';
    let useModelId = '';

    for (const provider of providers) {
      if (provider.id !== useProviderID) continue;
      const modelEntries = Object.entries(provider.models);
      if (modelEntries.length > 0) {
        useModelId = modelEntries[0][0];
        break;
      }
    }

    if (!useModelId) {
      throw new Error(`No model found for provider ${useProviderID}`);
    }

    debugLog(`[CodeAgent] Provider: ${useProviderID}, Model: ${useModelId}`);

    // 2. 创建新会话
    const sessionResult = await codeagentClient.session.create({});
    if (sessionResult.error) {
      throw new Error(sessionResult.error);
    }
    const sessionId = sessionResult.data?.id;
    if (!sessionId) {
      throw new Error('Empty session ID');
    }

    debugLog(`[CodeAgent] Session: ${sessionId}`);

    // 3. 启动权限监听（后台运行）
    const permissionPromise = (async () => {
      try {
        const { stream } = await codeagentClient!.event.subscribe();
        if (stream) {
          for await (const event of stream) {
            const evt = event as { type?: string; properties?: { id?: string; permission?: string } };
            // 排除 delta 消息，只打印其他事件
            if (evt?.type && !evt.type.includes('delta')) {
              debugLog(`[CodeAgent Event] ${JSON.stringify(evt)}`);
            }
            if (evt?.type === 'permission.asked') {
              const { id } = evt.properties || {};
              if (id && id.startsWith('per')) {
                await codeagentClient!.permission.reply({
                  requestID: id,
                  reply: 'always',
                });
                debugLog(`[CodeAgent] 权限已批准: ${id}`);
              }
            }
          }
        }
      } catch {
        // 忽略权限监听错误
      }
    })();

    // 4. 发送消息
    const response = await codeagentClient.session.prompt({
      sessionID: sessionId,
      model: {
        providerID: useProviderID,
        modelID: useModelId,
      },
      parts: [{ type: 'text', text: prompt }],
    });

    // 5. 提取响应文本
    const aiResponseText = response?.data?.parts
      ?.filter((part: unknown) => (part as { type?: string }).type === 'text')
      ?.map((part: unknown) => (part as { text?: string }).text || '')
      ?.join('');

    debugLog(`[CodeAgent] 响应: ${aiResponseText}`);

    return aiResponseText || '';
  } catch (e) {
    throw new Error(`CodeAgent SDK 调用错误: ${e}`);
  }
}

/**
 * 使用 Claude SDK 调用
 */
async function callClaudeAsync(prompt: string): Promise<string> {
  if (!startup) {
    throw new Error("Claude SDK 未安装，请运行 npm install @anthropic-ai/claude-agent-sdk@0.3.153");
  }

  try {
    // 预热 SDK（如果尚未预热）
    await warmupClaude();
    if (!warmInstance) {
      throw new Error("Claude SDK 预热失败");
    }

    const resultParts: string[] = [];

    // 使用 warm.query() 调用
    const queryResult = warmInstance.query(prompt);

    for await (const message of queryResult) {
      if (message.type === "result") {
        if (typeof message.result === 'string') {
          resultParts.push(message.result);
        }
      }
    }

    const result = resultParts.join("");
    return result;
  } catch (e) {
    throw new Error(`Claude SDK 调用错误: ${e}`);
  }
}

/**
 * 统一调用接口，根据环境变量选择 SDK
 */
async function callClaudeAsyncWrapper(prompt: string): Promise<string> {
  if (USE_CODEAGENT_SDK) {
    return await callCodeAgentAsync(prompt);
  } else {
    return await callClaudeAsync(prompt);
  }
}

// ==================== 决策逻辑 ====================

async function buildDecisionPrompt(
  todoList: TodoList,
  currentTodo: TodoItem,
  scheduleLogPath: string,
  workflowDocPath: string,
  reportPaths: Record<string, string[]>,
  todoListPath: string,
  todoDir: string
): Promise<string> {
  debugLog("[DEBUG] ===== build_decision_prompt 开始（并发模式）=====");

  const reviewStatus = currentTodo.review_status || {};
  const reviewStatusText = Object.entries(reviewStatus)
    .map(([k, v]) => `  - ${k}: ${v}`)
    .join("\n");

  // 支持 current_agent 为数组
  const currentAgents = Array.isArray(todoList.current_agent)
    ? todoList.current_agent
    : todoList.current_agent ? [todoList.current_agent] : [];

  const currentAgentsStr = currentAgents.length > 0 ? currentAgents.join(", ") : "(无)";
  debugLog(`[DEBUG] current_agent 数组: ${currentAgentsStr}`);
  debugLog(`[DEBUG] review_status: ${JSON.stringify(reviewStatus)}`);

  // 构建报告路径上下文
  let reportPathsText = "";

  // 包含所有 InProgress 审查的报告
  for (const [key, status] of Object.entries(reviewStatus)) {
    if (status === "InProgress") {
      const keyToReportKey: Record<string, string> = {
        "spec": "spec_reviewer",
        "code_quality": "quality_reviewer",
        "code_clean": "clean_reviewer",
        "code_security": "security_reviewer",
      };
      const reportKey = keyToReportKey[key];
      if (reportKey) {
        const pathList = reportPaths[reportKey] || [];
        for (const filePath of pathList) {
          if (fsSync.existsSync(filePath)) {
            debugLog(`[DEBUG] 文件 ${reportKey}: ${filePath}, 存在: True`);
            reportPathsText += `- ${reportKey}: ${filePath}\n`;
          }
        }
      }
    }
  }

  // 计算 retry 信息
  const retryCount = currentTodo.retry_count || 0;
  const maxRetry = currentTodo.max_retry || 3;

  // 检查是否有下一个 Todo
  const todos = todoList.todos || [];
  const currentIndex = todos.findIndex(t => t.id === currentTodo.id);
  const hasNextTodo = currentIndex >= 0 && currentIndex + 1 < todos.length;

  const decisionPrompt = `# 任务目标
请根据<上下文>，判断下一步操作。**本状态机支持并发执行多个审查任务**。

## 调度流程
- 审查完成后，分析所有报告，**只重审有问题的阶段**
- 所有审查通过 → nextTodo 或 finish

## 当前 Todo 情况
- ID: ${currentTodo.id || ''}
- 名称: ${currentTodo.name || ''}
- 上一次派发的 Agent: ${currentAgentsStr}
- 审查状态:
${reviewStatusText}
- 重试次数: ${retryCount}/${maxRetry}
- 是否有下一个 Todo: ${hasNextTodo ? "是" : "否"}

## 文件路径

### TodoList
${todoListPath}

### 工作产出报告
${reportPathsText || "(无工作产出报告)"}

## 决策要求

**重要约束**：你只负责判断下一步操作，不负责任何文件更新。文件更新由主Agent根据你的决策结果执行。

**审查状态说明**：
- \`Pending\`：尚未开始，需要启动
- \`InProgress\`：**审查尚未结束**，需要读取报告判断：
  - 场景1：Reviewer 已完成审查，需检查是否还有问题存在
  - 场景2：Implementer 已完成修改，需 Reviewer 重新审查是否已修复
- \`Passed\`：已完成且通过，无需重审

**判断逻辑**：
- **仅读取 review_status 中 InProgress 状态的审查报告**，Passed 状态保持信任
- 分析每个 InProgress 审查结果，构建 review_status_update：
  - 如果 Reviewer 已完成且无问题 → review_status_update[reviewType] = "Passed"
  - 如果 Reviewer 已完成但有问题 → 加入 failed_revs 列表，review_status_update[reviewType] = "InProgress"
- 检查重试次数：\`currentTodo.retry_count < currentTodo.max_retry\`（默认 max_retry = 3）
- 如果有 failed_revs：
  - 如果 retry_count < max_retry：
    - decision = "do-implement-fix"
    - parallel_tasks 包含 implementer 的修复任务
    - **review_status_update 包含本次已通过的审查（Passed）**，有问题的保持 InProgress
  - 如果 retry_count >= max_retry：
    - 达到最大重试次数，decision = "finish"
- 如果没有 failed_revs 且所有审查都已 Passed：
  - 还有下一个 Todo → decision = "nextTodo"
  - 没有下一个 Todo → decision = "finish"

## 输出格式
\`\`\`json
{
  "action": "dispatch_parallel",
  "decision": "do-parallel-review",
  "reason": "决策理由",
  "parallel_tasks": [
    {
      "action": "dispatch_spec_reviewer",
      "decision": "do-spec-review",
      "review_type": "spec"
    }
  ],
  "review_status_update": {
    "spec": "Passed",
    "code_quality": "Passed",
    "code_clean": "InProgress"
  }
}
\`\`\`

**字段说明**：
- \`decision\`：决策结果，枚举值见下方对照表
- \`parallel_tasks\`：需要并发执行的任务列表
- \`review_status_update\`：审查状态更新映射（key: review_type, value: "Passed" | "InProgress"）

**decision 枚举值**：
- \`do-parallel-review\`：并发派发 reviewer 或 implementer 重审
- \`do-implement-fix\`：派发 implementer 修复问题
- \`nextTodo\`：进入下一个 Todo
- \`finish\`：全部完成

**action 枚举值**：
- "dispatch_parallel"：并发派发多个 SubAgent
- "dispatch_implementer"：派发单个 implementer
- "nextTodo"：进入下一个 Todo
- "finish"：所有任务完成

**review_type 枚举**：
- "spec"：规格合规审查
- "code_quality"：代码质量审查
- "code_clean"：代码清洁审查
- "code_security"：代码安全审查
`;
  return decisionPrompt;
}

// ==================== 决策辅助函数 ====================

/**
 * 从 review_status 中获取待执行的审查类型列表
 * 状态为 Pending 或 InProgress 的审查都需要执行
 */
function getPendingOrInProgressReviews(reviewStatus: Record<string, string>): string[] {
  return Object.entries(reviewStatus)
    .filter(([_, status]) => status === "Pending" || status === "InProgress")
    .map(([type, _]) => type);
}

/**
 * 从 review_type 列表生成简化版 parallel_tasks（不含 prompt/report_path）
 * 用于 makeDecision 优化路径和断点恢复
 */
function buildSimplifiedParallelTasks(reviewTypes: string[]): ParallelTask[] {
  return reviewTypes.map(reviewType => ({
    action: REVIEW_TYPE_TO_ACTION[reviewType] || `dispatch_${reviewType}_reviewer`,
    decision: REVIEW_TYPE_TO_DECISION[reviewType] || `do-${reviewType}-review`,
    prompt: "",
    report_path: "",
    review_type: reviewType
  }));
}

/**
 * 判断 current_agent 是否只有 implementer
 */
function isImplementerOnly(currentAgents: string[]): boolean {
  return currentAgents.length === 1 && currentAgents[0] === "implementer";
}

async function makeDecision(
  todoList: TodoList,
  currentTodo: TodoItem,
  scheduleLogPath: string,
  workflowDocPath: string,
  reportPaths: Record<string, string[]>,
  todoListPath: string,
  todoDir: string
): Promise<DecisionResult> {
  debugLog("[DEBUG] ===== make_decision 开始（并发模式）=====");

  // 支持 current_agent 为数组
  const currentAgents = Array.isArray(todoList.current_agent)
    ? todoList.current_agent
    : todoList.current_agent ? [todoList.current_agent] : [];
  const reviewStatus = currentTodo.review_status || {};

  // ============================================================
  // 优化路径1：INIT 状态 → 直接派发 implementer
  // ============================================================
  if (currentTodo.status === "INIT") {
    debugLog("[DEBUG] 优化路径：todo 状态为 INIT，直接派发 implementer");

    return {
      action: "dispatch_implementer",
      decision: "do-implement",
      reason: "Todo 状态为 INIT，需要派发 implementer 开始实现",
      prompt: "",
      message: "",
      error: null,
      parallel_tasks: [{
        action: "dispatch_implementer",
        decision: "do-implement",
        prompt: "",
        report_path: "",
        review_type: ""
      }],
      review_status_update: {}
    };
  }

  // ============================================================
  // 优化路径2：implementer 完成 → 直接派发 pending/in-progress reviewers
  // ============================================================
  if (isImplementerOnly(currentAgents)) {
    const pendingReviews = getPendingOrInProgressReviews(reviewStatus);

    if (pendingReviews.length > 0) {
      debugLog(`[DEBUG] 优化路径：implementer 完成，直接派发 ${pendingReviews.length} 个 pending/in-progress reviewers`);

      const parallelTasks = buildSimplifiedParallelTasks(pendingReviews);

      debugLog("[DEBUG] ===== make_decision 结束（优化路径）=====");
      return {
        action: "dispatch_parallel",
        decision: "do-parallel-review",
        reason: `实现完成，触发 ${pendingReviews.length} 个 Pending/InProgress 审查并发执行`,
        prompt: "",
        message: "",
        error: null,
        parallel_tasks: parallelTasks,
        review_status_update: Object.fromEntries(
          pendingReviews.map(type => [type, "InProgress"])
        )
      };
    }

    // 所有审查都已完成（无 Pending/InProgress），直接判断 nextTodo 或 finish
    debugLog("[DEBUG] 优化路径：所有审查已完成，直接判断 nextTodo 或 finish");
    const todos = todoList.todos || [];
    const currentIndex = todos.findIndex(t => t.id === currentTodo.id);
    const hasNextTodo = currentIndex >= 0 && currentIndex + 1 < todos.length;

    debugLog("[DEBUG] ===== make_decision 结束（优化路径）=====");
    return {
      action: hasNextTodo ? "nextTodo" : "finish",
      decision: hasNextTodo ? "nextTodo" : "finish",
      reason: hasNextTodo ? "所有审查已完成，进入下一个 Todo" : "所有审查已完成，且无下一个 Todo",
      prompt: "",
      message: "",
      error: null,
      parallel_tasks: [],
      review_status_update: {}
    };
  }

  // ============================================================
  // 原有逻辑：调用大模型进行决策
  // ============================================================
  const decisionPrompt = await buildDecisionPrompt(
    todoList, currentTodo, scheduleLogPath, workflowDocPath, reportPaths, todoListPath, todoDir
  );
  debugLog(`[DEBUG] decision_prompt长度: ${decisionPrompt.length}`);

  debugLog("[DEBUG] 调用 Claude 进行决策...");
  const result = await callClaudeAsyncWrapper(decisionPrompt);
  debugLog(`[DEBUG] Claude返回结果长度: ${result.length}`);

  try {
    const jsonMatch = result.match(/\{[\s\S]*\}/);
    if (jsonMatch) {
      const decisionResult = JSON.parse(jsonMatch[0]);
      const decision = decisionResult.decision || "";

      debugLog(`[DEBUG] 解析decision: ${decision}`);

      // 有效的 decision 枚举值
      const validDecisions = ["do-implement", "do-parallel-review", "do-implement-fix", "nextTodo", "finish"];

      // 检查 decision 是否在有效范围内
      if (!validDecisions.includes(decision)) {
        debugLog(`[DEBUG] decision不在有效范围内: ${decision}`);
        debugLog("[DEBUG] ===== make_decision 结束 =====");
        return {
          action: "retry_state_machine",
          decision: decision,
          reason: `决策 '${decision}' 不在有效范围内`,
          prompt: "",
          message: "决策出现异常，请重试执行状态机",
          error: null
        };
      }

      const action = DECISION_TO_ACTION[decision] || "retry_state_machine";
      debugLog(`[DEBUG] 转换后的action: ${action}`);

      // 解析 parallel_tasks
      let parallelTasks: ParallelTask[] = [];
      if (decisionResult.parallel_tasks && Array.isArray(decisionResult.parallel_tasks)) {
        parallelTasks = decisionResult.parallel_tasks.map((task: any) => ({
          action: task.action || "",
          decision: task.decision || "",
          prompt: task.prompt || "",
          report_path: task.report_path || "",
          review_type: task.review_type || "",
        }));
        debugLog(`[DEBUG] 解析到 parallel_tasks: ${parallelTasks.length} 个任务`);
      }

      // 解析 review_status_update
      const reviewStatusUpdate = decisionResult.review_status_update || {};
      debugLog(`[DEBUG] review_status_update: ${JSON.stringify(reviewStatusUpdate)}`);

      debugLog("[DEBUG] ===== make_decision 结束 =====");
      return {
        action: action,
        decision: decision,
        reason: decisionResult.reason || "",
        prompt: "",
        message: "",
        error: null,
        parallel_tasks: parallelTasks,
        review_status_update: reviewStatusUpdate,
      };
    }
  } catch (e) {
    debugLog(`[DEBUG] make_decision - 解析失败: ${e}`);
  }

  debugLog(`[DEBUG] make_decision - 决策解析失败，返回error`);
  return {
    action: "error",
    decision: "error",
    reason: `无法解析决策结果: ${result.substring(0, 200)}`,
    prompt: "",
    message: "决策解析失败",
    error: "无法解析决策结果"
  };
}

async function generatePrompt(
  decision: string,
  currentTodo: TodoItem,
  todoList: TodoList,
  todoListPath: string,
  reportPaths: Record<string, string[]> = {}
): Promise<string> {
  debugLog(`[DEBUG] generate_prompt - decision: ${decision}`);

  // 1. 确定模板 key
  const templateKey = decision === "nextTodo" ? "implementer" : (DECISION_TO_TEMPLATE[decision] || "");
  if (!templateKey) {
    debugLog(`[DEBUG] generate_prompt - 未找到模板，返回空`);
    return "";
  }

  // 2. 读取并处理模板
  const template = await readTemplate(templateKey);
  let content = extractTemplateBody(template);

  // 3. 提取变量
  const planFile = todoList.plan_file || "";
  const planName = planFile ? path.basename(planFile, path.extname(planFile)) : "";
  const changeName = extractChangeName(planFile, todoListPath);
  const workingDirectory = todoList.working_directory || ".";
  const todoInfo = getCurrentTodoInfo(todoList, decision, currentTodo);

  // 4. 根据模板类型执行特定替换
  if (templateKey === "implementer") {
    // === Implementer 模板处理 ===

    // 4.1 获取 reportFile (fix 场景)
    let reportFile = "";
    if (decision.startsWith("fix-") || decision === "do-implement-fix") {
      // review_status 的 key 是 review_type，需要映射到 reportKey
      const reviewTypeToStatusKey: Record<string, string> = {
        "spec": "spec_reviewer",
        "code_quality": "quality_reviewer",
        "code_clean": "clean_reviewer",
        "code_security": "security_reviewer",
      };

      // 通过 review_status 获取状态为 InProgress 的 reviewers
      const reviewStatus = currentTodo.review_status || {};
      const reportFiles: string[] = [];

      for (const [reviewType, status] of Object.entries(reviewStatus)) {
        // InProgress 表示正在审查但还未完成（可能有失败结果）
        if (status === "InProgress") {
          const reportKey = reviewTypeToStatusKey[reviewType];
          if (reportKey) {
            const reportPath = getLatestReportFileByType(reportKey, reportPaths);
            if (reportPath) {
              reportFiles.push(reportPath);
            }
          }
        }
      }

      if (reportFiles.length > 0) {
        reportFile = reportFiles.map(p => `- ${p}`).join("\n");
      }
    }

    // 4.2 构建并替换 TaskDescription
    const taskDescription = buildImplementerTaskDescription(decision, {
      workingDirectory,
      planFile,
      currentDescription: todoInfo.description,
      currentContext: todoInfo.context,
      reportFile,
    });
    content = content.replace(/\$\{TaskDescription\}/g, taskDescription);

  } else {
    // === Reviewer 模板处理 ===

    // 4.3 获取 implementer 报告列表
    const implementerReportPaths = reportPaths["implementer"] || [];
    const implementerReports = getImplementerReportsAsList(implementerReportPaths);

    // 4.4 构建并替换 ReviewScope
    const reviewScope = buildReviewerScope(implementerReports);
    content = content
      .replace(/\$\{TaskDescription\}/g, todoInfo.description)
      .replace(/\$\{ReviewScope\}/g, reviewScope);
  }

  // 5. 执行公共替换
  content = content
    .replace(/\$\{CLAUDE_PROJECT_DIR\}/g, workingDirectory)
    .replace(/<change-name>/g, changeName)
    .replace(/<plan-name>\/<todo-name>/g, `${planName}/${todoInfo.name}`)
    .replace(/\$\{todoTitle\}/g, todoInfo.title);

  debugLog(`[DEBUG] generate_prompt - 完成，生成长度: ${content.length}`);
  return content;
}

// ==================== 并发决策相关函数 ====================

/**
 * 根据 Agent 类型获取对应的决策
 */
function getDecisionByAgentType(agentType: string): string {
  return AGENT_TYPE_TO_DECISION[agentType] || "";
}

/**
 * 获取报告文件路径
 */
function getReportPathByAgentType(agentType: string, todoDir: string): string {
  return path.join(todoDir, AGENT_TO_DEFAULT_FILENAME[agentType] || "");
}

/**
 * 获取审查类型
 */
function getReviewTypeByAgentType(agentType: string): string {
  // implementer 没有审查类型
  if (agentType === "implementer") return "";

  const reviewTypeMap: Record<string, string> = {
    "spec_reviewer": "spec",
    "quality_reviewer": "code_quality",
    "clean_reviewer": "code_clean",
    "security_reviewer": "code_security",
  };
  return reviewTypeMap[agentType] || agentType;
}

/**
 * 生成并发任务的提示词
 */
async function generateParallelPrompts(
  decisions: string[],
  currentTodo: TodoItem,
  todoList: TodoList,
  todoListPath: string,
  todoDir: string,
  reportPaths: Record<string, string[]>
): Promise<ParallelTask[]> {
  const tasks: ParallelTask[] = [];

  for (const decision of decisions) {
    const prompt = await generatePrompt(decision, currentTodo, todoList, todoListPath, reportPaths);
    const templateKey = DECISION_TO_TEMPLATE[decision] || "";
    const reportPath = getReportPathByAgentType(templateKey, todoDir);
    const reviewType = getReviewTypeByAgentType(templateKey);

    // 构建 action
    const actionMap: Record<string, string> = {
      "implementer": "dispatch_implementer",
      "spec_reviewer": "dispatch_spec_reviewer",
      "quality_reviewer": "dispatch_quality_reviewer",
      "clean_reviewer": "dispatch_clean_reviewer",
      "security_reviewer": "dispatch_security_reviewer",
    };
    const action = actionMap[templateKey] || `dispatch_${templateKey}`;

    tasks.push({
      action: action,
      decision: decision,
      prompt: prompt,
      report_path: reportPath,
      review_type: reviewType,
    });
  }

  return tasks;
}

/**
 * 根据 decision 结果更新 todoList
 * 这是状态机的核心职责，todoList.json 的更新必须由状态机完成
 */
async function applyDecisionToTodoList(
    todoList: TodoList,
    todoListPath: string,
  currentTodo: TodoItem,
  decisionResult: DecisionResult
): Promise<void> {
  const decision = decisionResult.decision;
  const parallelTasks = decisionResult.parallel_tasks || [];
  const reviewStatusUpdate = decisionResult.review_status_update || {};

  switch (decision) {
    case "do-implement":
      // 派发 implementer（Step-1 初始实现）
      todoList.current_agent = ["implementer"];
      currentTodo.status = "RUNNING";
      break;

    case "do-parallel-review":
      // 派发多个 reviewer 并发执行（Step-2 首次派发，或 Step-3 重审）
      todoList.current_agent = parallelTasks.map(t => {
        // 从 action "dispatch_spec_reviewer" 提取 "spec_reviewer"
        const reviewType = t.review_type;
        const agentTypeMap: Record<string, string> = {
          "spec": "spec_reviewer",
          "code_quality": "quality_reviewer",
          "code_clean": "clean_reviewer",
          "code_security": "security_reviewer",
        };
        return agentTypeMap[reviewType] || reviewType;
      });

      // 更新 review_status: Pending → InProgress
      for (const task of parallelTasks) {
        if (currentTodo.review_status && currentTodo.review_status[task.review_type] === "Pending") {
          currentTodo.review_status[task.review_type] = "InProgress";
        }
      }
      break;

    case "do-implement-fix":
      // implementer 修复问题（Step-3 有 failed_revs 时触发）
      todoList.current_agent = ["implementer"];
      // 重试次数 +1（implementer 修复 + reviewer 重审 = 一轮）
      currentTodo.retry_count = (currentTodo.retry_count || 0) + 1;

      // 根据 review_status_update 更新状态
      if (reviewStatusUpdate) {
        for (const [reviewType, status] of Object.entries(reviewStatusUpdate)) {
          if (status == 'Passed') {
            currentTodo.review_status[reviewType] = status;
          }
        }
      }

      // 如果 retry_count >= max_retry，强制标记为需要人工介入
      if (currentTodo.retry_count >= (currentTodo.max_retry || 3)) {
        currentTodo.status = "finished_with_warnings";
      }
      break;

    case "nextTodo":
      // 进入下一个 Todo（Step-3 所有审查通过，有下一个 Todo）
      todoList.current_agent = ["implementer"];
      currentTodo.status = "COMPLETED";

      // 重置 retry_count（进入下一个 Todo）
      currentTodo.retry_count = 0;

      // 将当前 Todo 的已存在 review_status 全部刷新为 Passed（只更新已存在的 Key）
      if (currentTodo.review_status) {
        for (const key of Object.keys(currentTodo.review_status)) {
          currentTodo.review_status[key] = "Passed";
        }
      }

      // 更新 current_todo_index
      todoList.current_todo_index = (todoList.current_todo_index || 0) + 1;

      // 将下一个 Todo 设为 RUNNING
      const nextTodo = todoList.todos[todoList.current_todo_index];
      if (nextTodo) {
        nextTodo.status = "RUNNING";
      }
      break;

    case "finish":
      // 全部完成（Step-3 所有审查通过，无下一个 Todo）
      todoList.current_agent = [];
      currentTodo.status = "finished";

      // 根据 review_status_update 更新状态
      if (reviewStatusUpdate) {
        for (const [reviewType, status] of Object.entries(reviewStatusUpdate)) {
          if (currentTodo.review_status) {
            currentTodo.review_status[reviewType] = status;
          }
        }
      }
      break;
  }

  todoList.updated_at = toLocalISOString();
  // 优先使用 CLAUDE_CODE_SESSION_ID（Claude Code 场景），其次使用 CODEAGENT_SESSION_ID（CodeAgent 场景）
  todoList.session_id = process.env.CLAUDE_CODE_SESSION_ID || process.env.CODEAGENT_SESSION_ID;
  await fs.writeFile(todoListPath, JSON.stringify(todoList, null, 2), 'utf-8');
  debugLog(`[DEBUG] applyDecisionToTodoList - decision: ${decision}, 更新完成, session_id: ${todoList.session_id}`);
}

// ==================== 主处理逻辑 ====================

async function processStateMachine(todoListPath: string): Promise<DecisionResult> {
  debugLog("[DEBUG] ===== process_state_machine 开始（并发模式）=====");
  debugLog(`[DEBUG] todoList路径: ${todoListPath}`);

  const todoList = await readTodoList(todoListPath);
  const currentTodoId = todoList.current_todo_id || "";

  // 支持 current_agent 为数组
  const currentAgents = Array.isArray(todoList.current_agent)
    ? todoList.current_agent
    : todoList.current_agent ? [todoList.current_agent] : [];

  debugLog(`[DEBUG] current_todo_id: ${currentTodoId}`);
  debugLog(`[DEBUG] current_agent 数组: ${JSON.stringify(currentAgents)}`);
  debugLog(`[DEBUG] todos数量: ${(todoList.todos || []).length}`);

  // 获取当前 Todo（优先使用 current_todo_index）
  let currentTodo: TodoItem | undefined;
  const currentIndex = todoList.current_todo_index ?? 0;
  if (todoList.todos && todoList.todos[currentIndex]) {
    currentTodo = todoList.todos[currentIndex];
  }
  if (!currentTodo) {
    currentTodo = (todoList.todos || []).find(t => t.id === currentTodoId);
  }

  if (!currentTodo) {
    debugLog(`[DEBUG] 未找到当前todo: ${currentTodoId}`);
    return {
      action: "error",
      decision: "error",
      reason: `未找到当前 todo: ${currentTodoId}`,
      prompt: "",
      message: `未找到当前 todo: ${currentTodoId}`,
      error: `未找到当前 todo: ${currentTodoId}`
    };
  }

  debugLog(`[DEBUG] 当前todo名称: ${currentTodo.name || ''}`);
  debugLog(`[DEBUG] 当前todo状态: ${currentTodo.status || ''}`);
  debugLog(`[DEBUG] 当前todo的review_status: ${JSON.stringify(currentTodo.review_status || {})}`);

  // 获取调度日志和流程图路径
  const todoListDir = path.dirname(todoListPath);
  const scheduleLogPath = path.join(todoListDir, "scheduleLog.md");
  const workflowDocPath = path.join(getDocsDir(), "workflow.md");

  debugLog(`[DEBUG] todo目录: ${todoListDir}`);
  debugLog(`[DEBUG] 调度日志路径: ${scheduleLogPath}`);

  // 获取当前 todo 的报告文件路径（含容错机制）
  const todoDir = path.join(todoListDir, sanitizeFolderName(currentTodo.name || ""));
  const reportPaths = buildReportPaths(todoDir, currentTodo.review_status, currentAgents[0] || "");
  debugLog(`[DEBUG] 报告文件路径: ${JSON.stringify(reportPaths)}`);

  // ==================== 断点恢复检测 ====================
  const breakpointCheck = await checkBreakpointRecovery(todoList, todoDir, todoListPath, reportPaths);
  if (breakpointCheck.need_recovery) {
    const decision = breakpointCheck.decision || "";
    const reason = breakpointCheck.reason || "";

    debugLog(`[DEBUG] 断点恢复检测 - decision: ${decision}, reason: ${reason.substring(0, 100)}...`);

    // 如果是 finish 或 regenerate，直接返回
    if (decision === "finish" || decision === "regenerate") {
      return {
        action: decision,
        decision: decision,
        reason: reason,
        prompt: "",
        message: reason,
        is_breakpoint_recovery: true,
        error: null
      };
    }

    // 断点恢复场景：返回 parallel_tasks
    if (breakpointCheck.parallel_tasks && breakpointCheck.parallel_tasks.length > 0) {
      return {
        action: breakpointCheck.action,
        decision: breakpointCheck.decision,
        reason: breakpointCheck.reason,
        prompt: "",
        message: breakpointCheck.message,
        is_breakpoint_recovery: true,
        parallel_tasks: breakpointCheck.parallel_tasks,
        error: null
      };
    }
  }
  // ====================

  // ==================== 正常决策流程 ====================
  // 调用 Claude 进行决策
  debugLog("[DEBUG] 调用 make_decision 进行决策...");
  const decisionResult = await makeDecision(
    todoList, currentTodo, scheduleLogPath, workflowDocPath, reportPaths, todoListPath, todoDir
  );

  const action = decisionResult.action;
  const decision = decisionResult.decision;
  const reason = decisionResult.reason;
  const parallelTasks = decisionResult.parallel_tasks || [];
  const reviewStatusUpdate = decisionResult.review_status_update || {};

  debugLog(`[DEBUG] 决策结果 - decision: ${decision}, action: ${action}`);
  debugLog(`[DEBUG] 决策理由: ${reason.substring(0, 100)}...`);
  debugLog(`[DEBUG] parallel_tasks 数量: ${parallelTasks.length}`);

  // 根据 decision 结果更新 todoList（核心约束：状态机负责更新）
  await applyDecisionToTodoList(todoList, todoListPath, currentTodo, decisionResult);

  // 如果需要派发任务，生成 parallel_tasks 的提示词
  let finalParallelTasks: ParallelTask[] = [];
  if (action === "dispatch_parallel" || decision === "do-parallel-review" || decision === "do-implement-fix" || decision === "do-implement" || decision === "nextTodo" || action === "dispatch_implementer") {
    if (decision === "nextTodo") {
      // nextTodo 时派发下一个 Todo 的 implementer
      const prompt = await generatePrompt("do-implement", currentTodo, todoList, todoListPath, reportPaths);
      finalParallelTasks = [{
        action: "dispatch_implementer",
        decision: "do-implement",
        prompt: prompt,
        report_path: path.join(todoDir, AGENT_TO_DEFAULT_FILENAME["implementer"]),
        review_type: "",
      }];
    } else if (decision === "do-implement-fix" || decision === "do-implement") {
      // 单个 implementer 任务
      const prompt = await generatePrompt(decision, currentTodo, todoList, todoListPath, reportPaths);
      finalParallelTasks = [{
        action: "dispatch_implementer",
        decision: decision,
        prompt: prompt,
        report_path: path.join(todoDir, AGENT_TO_DEFAULT_FILENAME["implementer"]),
        review_type: "",
      }];
    } else if (parallelTasks.length > 0) {
      // parallel_tasks 中缺少 prompt 和 report_path，需要调用 generateParallelPrompts 补充
      const decisions = parallelTasks.map(t => t.decision);
      finalParallelTasks = await generateParallelPrompts(
        decisions,
        currentTodo,
        todoList,
        todoListPath,
        todoDir,
        reportPaths
      );
    }  else {
      // 需要根据 review_status 中的 Pending 状态生成
      const pendingReviews: string[] = [];
      const reviewStatus = currentTodo.review_status || {};
      for (const [key, status] of Object.entries(reviewStatus)) {
        if (status === "Pending") {
          const reviewTypeMap: Record<string, string> = {
            "spec": "spec_reviewer",
            "code_quality": "quality_reviewer",
            "code_clean": "clean_reviewer",
            "code_security": "security_reviewer",
          };
          const agentType = reviewTypeMap[key];
          if (agentType) {
            pendingReviews.push(agentType);
          }
        }
      }

      if (pendingReviews.length > 0) {
        finalParallelTasks = await generateParallelPrompts(
          pendingReviews.map(type => getDecisionByAgentType(type)),
          currentTodo,
          todoList,
          todoListPath,
          todoDir,
          reportPaths
        );
      }
    }
  }

  // 生成用户友好的消息
  let message = reason;
  let finalAction = action;
  let finalDecision = decision;

  if (finalParallelTasks.length > 0) {
    const reviewTypes = finalParallelTasks.map(t => t.review_type || t.action.replace("dispatch_", "")).join(", ");
    message = `派发 ${finalParallelTasks.length} 个 Agent 并发执行: ${reviewTypes}`;
    // nextTodo 时 action 和 decision 都覆盖为与首次 implement 一致
    if (decision === "nextTodo") {
      finalAction = "dispatch_implementer";
      finalDecision = "do-implement";
      message = `[nextTodo] 派发 ${finalParallelTasks.length} 个 Agent 并发执行: ${reviewTypes}`;
    }
  }

  debugLog(`[DEBUG] ===== process_state_machine 结束 =====`);
  return {
    action: finalAction,
    decision: finalDecision,
    reason: reason,
    prompt: "",
    message: message,
    critical_hint: CRITICAL_HINT,
    error: null,
    parallel_tasks: finalParallelTasks,
    review_status_update: reviewStatusUpdate,
  };
}

// ==================== 主入口 ====================

function findCommonParent(path1: string, path2: string): string {
  // 解析路径，获取各部分（只取目录部分，不包含文件名）
  const parsed1 = path.parse(path1);
  const parsed2 = path.parse(path2);

  // 使用 path.parts 等价方式获取路径部分
  // 只取目录部分，与 Python 的 Path.parts 一致（包含驱动器）
  const parts1 = parsed1.dir ? parsed1.dir.split(path.sep).filter(Boolean) : [];
  const parts2 = parsed2.dir ? parsed2.dir.split(path.sep).filter(Boolean) : [];

  const commonParts: string[] = [];
  for (let i = 0; i < Math.min(parts1.length, parts2.length); i++) {
    if (parts1[i].toLowerCase() === parts2[i].toLowerCase()) {
      commonParts.push(parts1[i]);
    } else {
      break;
    }
  }

  if (!commonParts.length) {
    // 返回驱动器或根路径
    return parsed1.root || parsed1.dir.split(path.sep)[0] || "";
  }

  // 返回时使用 path1 的风格（保留原始的大小写形式）
  return path.join(...commonParts);
}

function fixWorkingDirectory(todoList: TodoList): FixWorkingDirectoryResult {
  const workingDirectory = todoList.working_directory || "";
  const planFile = todoList.plan_file || "";

  if (!workingDirectory || !planFile) {
    return {
      fixed: false,
      old_working_directory: workingDirectory,
      working_directory: workingDirectory,
      reason: "working_directory 或 plan_file 为空，无需修复"
    };
  }

  const workingPath = path.resolve(workingDirectory);
  const planPath = path.resolve(planFile);

  // 检查 plan_file 是否存在
  if (!fsSync.existsSync(planPath)) {
    return {
      fixed: false,
      old_working_directory: workingDirectory,
      working_directory: workingDirectory,
      reason: `plan_file 不存在: ${planFile}，无需修复`
    };
  }

  // 检查 working_directory 是否是 plan_file 的祖宗目录
  try {
    const workingResolved = path.resolve(workingPath);
    const planResolved = path.resolve(planPath);

    // 检查 plan_file 是否在 working_directory 的子树中
    const relativePath = path.relative(workingResolved, planResolved);
    const isChild = !relativePath.startsWith('..') && !path.isAbsolute(relativePath);

    if (isChild) {
      return {
        fixed: false,
        old_working_directory: workingDirectory,
        working_directory: workingDirectory,
        reason: "working_directory 已是 plan_file 的祖宗目录，无需修复"
      };
    }
    // plan_file 不在 working_directory 子树中，需要修复
  } catch (e) {
    return {
      fixed: false,
      old_working_directory: workingDirectory,
      working_directory: workingDirectory,
      reason: `路径解析失败: ${e}，无需修复`
    };
  }

  // 计算公共父目录
  const commonParent = findCommonParent(workingPath, planPath);
  const newWorkingDirectory = commonParent;

  return {
    fixed: true,
    old_working_directory: workingDirectory,
    working_directory: newWorkingDirectory,
    reason: `working_directory (${workingDirectory}) 不是 plan_file (${planFile}) 的祖宗目录，已修复为公共父目录: ${newWorkingDirectory}`
  };
}

// ==================== 命令行入口 ====================

async function main(): Promise<void> {
  const args = process.argv.slice(2);

  let statePath = "";
  let debugEnabled = false;

  for (let i = 0; i < args.length; i++) {
    if (args[i] === "--state" && i + 1 < args.length) {
      statePath = args[i + 1];
      i++;
    } else if (args[i] === "--debug") {
      debugEnabled = true;
    } else if (!args[i].startsWith("--")) {
      statePath = args[i];
    }
  }

  if (!statePath) {
    console.error("用法: state_machine.ts --state <todoList.json路径> [--debug]");
    process.exit(1);
  }

  DEBUG_MODE = debugEnabled;

  // 将相对路径转换为绝对路径
  if (!path.isAbsolute(statePath)) {
    statePath = path.resolve(statePath);
    debugLog(`[DEBUG] 相对路径已转换为绝对路径: ${statePath}`);
  }

  // 预先读取 todolist 文件，修复 working_directory 并计算 plan_hash，同时记录 session_id
  try {
    const todoList = await readTodoList(statePath);

    // 优先使用 CLAUDE_CODE_SESSION_ID（Claude Code 场景），其次使用 CODEAGENT_SESSION_ID（CodeAgent 场景）
    const currentSessionId = process.env.CLAUDE_CODE_SESSION_ID || process.env.CODEAGENT_SESSION_ID;
    if (todoList.session_id !== currentSessionId) {
      todoList.session_id = currentSessionId;
      await writeTodoList(statePath, todoList);
      debugLog(`[DEBUG] 已更新 todoList.json 中的 session_id: ${currentSessionId}`);
    }

    // 修复 working_directory
    const fixResult = fixWorkingDirectory(todoList);

    if (fixResult.fixed) {
      debugLog(`[DEBUG] ${fixResult.reason}`);
      todoList.working_directory = fixResult.working_directory;
      await writeTodoList(statePath, todoList);
      debugLog(`[DEBUG] 已更新 todoList.json 中的 working_directory`);
    } else {
      debugLog(`[DEBUG] ${fixResult.reason}`);
    }

    // 如果 plan_hash 为空，则计算并写入
    if (!todoList.plan_hash && todoList.plan_file) {
      const computedHash = computePlanHash(todoList.plan_file);
      if (computedHash) {
        todoList.plan_hash = computedHash;
        await writeTodoList(statePath, todoList);
        debugLog(`[DEBUG] 已计算并写入 plan_hash: ${computedHash}`);
      }
    }
  } catch (e) {
    debugLog(`[DEBUG] 读取或修复 todoList 失败: ${e}，将继续使用原始路径`);
  }

  try {
    const result = await processStateMachine(statePath);
    result.critical_hint = CRITICAL_HINT;
    console.log(JSON.stringify(result, null, 2));
    process.exit(0);
  } catch (e) {
    const errorMsg = e instanceof Error ? e.message : String(e);
    console.log(JSON.stringify({
      action: "error",
      decision: "error",
      reason: errorMsg,
      prompt: "",
      message: errorMsg,
      error: errorMsg
    }, null, 2));
    process.exit(1);
  }
}

// 运行主函数
main().catch(e => {
  const errorMsg = e instanceof Error ? e.message : String(e);
  console.error("未捕获的错误:", errorMsg);
  process.exit(1);
});
