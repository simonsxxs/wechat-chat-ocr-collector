# collect.py 脚本全文

> 本文件是采集脚本的完整代码。如果你的环境里没有 `scripts/collect.py`（例如只拿到了纯 md 安装包），把下面代码块中的内容**原样保存**为工作目录下的 `collect.py` 即可，用法见 SKILL.md。
>
> 脚本只读屏幕（截图 + 本地离线 OCR + 窗口消息滚动），不碰数据库、不解密、不注入进程、不联网、不发消息。依赖：`pip install pywin32 pillow rapidocr-onnxruntime`

```python
# -*- coding: utf-8 -*-
"""
微信聊天记录全自动采集器（OCR 版 · 通用）
==========================================

原理
----
把微信窗口提到最前 → 循环【截图聊天区 → 本地 OCR 识别 → 用窗口消息向上滚动】
→ 直到翻到最早的位置 → 合并去重，输出纯文本。

为什么只能用这个方案（重要，别再走弯路）
----------------------------------------
1. 微信 4.x 的本地聊天数据库是 SQLCipher 加密的；过去能解密的主流工具
   （wx_key / PyWxDump / WeChatMsg）已被数据库官方以法务手段全部下架，
   解密属"绕过技术保护措施"，有法律风险。这条路不要走。
2. 很多受限环境禁止向其他进程注入键盘/鼠标输入，所以"模拟 Ctrl+A / Ctrl+C"
   这类复制方案在部分环境里根本送不进目标窗口。
3. 稳定可行的只有：**屏幕截图 + 本地离线 OCR**。
   全程只读屏幕：不碰数据库、不解密、不注入进程、不联网上传、不发任何消息。

适用范围
--------
· 微信 PC 版（Windows）
· 群聊、私聊都可以
· 唯一前提：**你要先手动在微信里点开目标聊天**（程序无法切换会话）

用法
----
    # 第 1 步：先校准，确认截图区域抓对了
    python collect.py --calibrate

    # 第 2 步：正式采集（把 target 换成目标窗口标题关键字）
    python collect.py --out "聊天记录.txt"

    # 若自动找窗口不准，可指定标题关键字
    python collect.py --title 微信 --out "聊天记录.txt"

依赖
----
    pip install pywin32 pillow rapidocr-onnxruntime

参数说明
--------
    --title       窗口标题包含的关键字，默认 微信；企业微信可传 企业微信
    --out         输出 txt 路径
    --crop-left   聊天区左边界比例（0~1），默认 0.30。校准后可调
    --crop-left-px 聊天区左边界固定像素（优先级高于比例，适合列表宽度固定的情况）
    --crop-top    顶部排除像素，默认 30
    --crop-bottom 底部排除像素（输入框区域），默认 210
    --wheel-per-step 每次滚动的滚轮格数，默认 12
    --wait        每轮滚动后等待秒数，默认 0.7
    --max-screens 最大屏数上限，默认 400
    --calibrate   仅截图 + OCR 一屏，用于确认区域

注意事项
--------
· 采集期间微信会被顶到最前，会挡住其他窗口，属正常现象。
· 采集期间不要动鼠标键盘，不要手动滚动聊天窗口。
· 请勿在采集过程中关闭微信。
"""
import sys, io, os, time, argparse
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8', errors='replace')

import ctypes
from ctypes import wintypes
from PIL import ImageGrab

u = ctypes.windll.user32
u.SetProcessDPIAware()

HWND_TOPMOST, HWND_NOTOPMOST = -1, -2
SWP_NOSIZE, SWP_NOMOVE, SWP_SHOWWINDOW = 0x0001, 0x0002, 0x0040
WM_MOUSEWHEEL = 0x020A


# ---------------------------------------------------------------- 窗口工具
def win_title(h):
    n = u.GetWindowTextLengthW(h)
    if n == 0:
        return ''
    b = ctypes.create_unicode_buffer(n + 1)
    u.GetWindowTextW(h, b, n + 1)
    return b.value


def win_rect(h):
    r = wintypes.RECT()
    u.GetWindowRect(h, ctypes.byref(r))
    return r.left, r.top, r.right, r.bottom


def find_main_window(keyword):
    """查找标题含 keyword、面积最大的可见窗口。返回全部候选。"""
    found = []
    CB = ctypes.WINFUNCTYPE(ctypes.c_bool, wintypes.HWND, wintypes.LPARAM)

    def cb(h, l):
        if not u.IsWindowVisible(h):
            return True
        t = win_title(h)
        if not t or keyword not in t:
            return True
        L, T, R, B = win_rect(h)
        w, hh = R - L, B - T
        if w < 400 or hh < 300:
            return True
        found.append((w * hh, h, (L, T, R, B), t))
        return True

    u.EnumWindows(CB(cb), 0)
    found.sort(reverse=True)
    return found


def enum_children(h, depth=0, acc=None):
    if acc is None:
        acc = []
    if depth > 4:
        return acc
    CB = ctypes.WINFUNCTYPE(ctypes.c_bool, wintypes.HWND, wintypes.LPARAM)
    kids = []

    def cb(ch, l):
        kids.append(ch); return True

    u.EnumChildWindows(h, CB(cb), 0)
    for k in kids:
        acc.append((k, win_title(k), win_rect(k)))
        enum_children(k, depth + 1, acc)
    return acc


def raise_window(hwnd):
    """把窗口提到最前（不抢输入焦点也能截图）"""
    try:
        u.ShowWindow(hwnd, 9)          # SW_RESTORE
    except Exception:
        pass
    time.sleep(0.2)
    u.SetWindowPos(hwnd, HWND_TOPMOST, 0, 0, 0, 0, SWP_NOSIZE | SWP_NOMOVE | SWP_SHOWWINDOW)
    time.sleep(0.25)
    u.SetWindowPos(hwnd, HWND_NOTOPMOST, 0, 0, 0, 0, SWP_NOSIZE | SWP_NOMOVE | SWP_SHOWWINDOW)
    time.sleep(0.2)
    try:
        u.BringWindowToTop(hwnd)
    except Exception:
        pass
    u.SetForegroundWindow(hwnd)
    time.sleep(0.7)
    return u.GetForegroundWindow() == hwnd


# ---------------------------------------------------------------- 主流程
def main():
    ap = argparse.ArgumentParser(description='微信聊天记录 OCR 采集器')
    ap.add_argument('--title', default='微信', help='窗口标题关键字，默认 微信')
    ap.add_argument('--hwnd', type=lambda v: int(v, 0), default=0, help='直接指定窗口句柄')
    ap.add_argument('--list', action='store_true', help='列出所有匹配的窗口后退出')
    ap.add_argument('--out', default='聊天记录.txt', help='输出 txt')
    ap.add_argument('--crop-left', type=float, default=0.30, help='聊天区左边界比例')
    ap.add_argument('--crop-left-px', type=int, default=0, help='聊天区左边界固定像素')
    ap.add_argument('--crop-top', type=int, default=30, help='顶部排除像素')
    ap.add_argument('--crop-bottom', type=int, default=210, help='底部排除像素')
    ap.add_argument('--wheel-per-step', type=int, default=12, help='每次滚动格数')
    ap.add_argument('--wait', type=float, default=0.7, help='滚动后等待秒数')
    ap.add_argument('--max-screens', type=int, default=400, help='最大屏数')
    ap.add_argument('--calibrate', action='store_true', help='只截图+OCR一屏，用于校准')
    args = ap.parse_args()

    # --- 找窗口 ---
    cands = find_main_window(args.title)
    if not cands:
        print(f'未找到标题含「{args.title}」的窗口。请先打开微信。')
        return 2

    if args.list:
        print(f'找到 {len(cands)} 个匹配窗口：')
        for i, (area, h, r, t) in enumerate(cands):
            print(f'  [{i}] hwnd={h}  rect={r}  尺寸={r[2]-r[0]}x{r[3]-r[1]}  标题={t!r}')
        print('\n用 --hwnd <句柄> 指定要采集的那个（例如开了多个微信账号时）。')
        return 0

    if args.hwnd:
        sel = [c for c in cands if c[1] == args.hwnd]
        if not sel:
            print(f'指定的 hwnd={args.hwnd} 不在匹配列表里，请用 --list 查看。')
            return 2
        hwnd, (L, T, R, B), title = sel[0][1], sel[0][2], sel[0][3]
    else:
        if len(cands) > 1:
            print(f'注意：找到 {len(cands)} 个匹配窗口，已默认选面积最大的那个。')
            for i, (area, h, r, t) in enumerate(cands):
                print(f'  [{i}] hwnd={h}  rect={r}')
            print('  如果选错了，用 --list 查看后用 --hwnd 指定。\n')
        hwnd, (L, T, R, B), title = cands[0][1], cands[0][2], cands[0][3]

    print(f'目标窗口: {title!r}  hwnd={hwnd}')
    print(f'窗口矩形: ({L},{T},{R},{B})')

    if not raise_window(hwnd):
        print('警告: 未能确认窗口已置前，仍继续尝试')

    # --- 找渲染子窗口（滚动要发给它）---
    ch = enum_children(hwnd)
    sub = [k for k, t, r in ch if t and ('Render' in t or 'MMUI' in t)]
    scroll_target = sub[0] if sub else hwnd
    SL, ST, SR, SB = win_rect(scroll_target)
    print(f'渲染窗口: hwnd={scroll_target} rect=({SL},{ST},{SR},{SB})')

    # --- 计算聊天区截图范围 ---
    if args.crop_left_px > 0:
        cx = SL + args.crop_left_px
    else:
        cx = SL + int((SR - SL) * args.crop_left)
    box = (cx, ST + args.crop_top, SR - 8, SB - args.crop_bottom)
    print(f'聊天区截图范围: {box}')

    # --- OCR 引擎 ---
    print('\n加载 OCR 引擎（首次会初始化模型）…')
    from rapidocr_onnxruntime import RapidOCR
    engine = RapidOCR()
    print('引擎就绪\n')

    # --- 校准模式 ---
    if args.calibrate:
        img = ImageGrab.grab(bbox=box, all_screens=True)
        p = os.path.abspath('_calibrate.png')
        img.save(p)
        print(f'已保存校准截图: {p}  (尺寸 {img.size})')
        res, _ = engine(p)
        print(f'\nOCR 识别到 {len(res) if res else 0} 行：')
        print('-' * 50)
        if res:
            for it in res:
                print('  ' + it[1])
        print('-' * 50)
        print('\n判断标准：')
        print('  · 如果这里看到的是【聊天内容】→ 区域正确，可以正式采集')
        print('  · 如果看到的是【左侧会话列表 / 好友名单】→ 调大 --crop-left（如 0.35）')
        print('  · 如果左右都混在一起 → 微调 --crop-left')
        print('  · 如果底部露出【输入框】→ 调大 --crop-bottom')
        return 0

    # --- 滚动消息用的 lParam（聊天区中心）---
    mx = cx + int((SR - cx) / 2)
    my = ST + int((SB - ST) * 0.4)
    lparam = ((my & 0xFFFF) << 16) | (mx & 0xFFFF)

    screens = []
    prev_lines = set()
    stall = 0
    t0 = time.time()
    tmp_png = os.path.abspath('_tmp_screen.png')

    for i in range(1, args.max_screens + 1):
        # 保证窗口在前面
        if u.GetForegroundWindow() != hwnd:
            if raise_window(hwnd):
                print(f'  (第 {i} 屏前已自动把窗口重新提到最前)', flush=True)
            else:
                print(f'  (第 {i} 屏前无法置前，跳过)', flush=True)
                time.sleep(1.0)
                continue

        img = ImageGrab.grab(bbox=box, all_screens=True)
        img.save(tmp_png)
        result, _ = engine(tmp_png)
        lines = [it[1].strip() for it in result if it and it[1] and it[1].strip()] if result else []

        cur = set(lines)
        # 停止判定：与上一屏内容重合度 > 0.9 视为"没动"
        if prev_lines and cur:
            inter = len(cur & prev_lines)
            union = len(cur | prev_lines) or 1
            ratio = inter / union
            if ratio > 0.9:
                stall += 1
            else:
                stall = 0
        prev_lines = cur

        if lines:
            screens.append(lines)

        print(f'[屏 {i:>3}] 识别 {len(lines):>3} 行 | 停滞 {stall} | 已收集 {len(screens)} 屏 | {time.time()-t0:.0f}s', flush=True)

        if stall >= 4:
            print('\n连续 4 屏内容不再变化 → 已到最早位置，停止。')
            break

        for _ in range(args.wheel_per_step):
            u.PostMessageW(scroll_target, WM_MOUSEWHEEL, (120 << 16), lparam)
            time.sleep(0.06)
        time.sleep(args.wait)

    try:
        os.remove(tmp_png)
    except Exception:
        pass

    # --- 合并去重（从最早的一屏开始排，保证时间顺序）---
    print('\n合并去重中…')
    seen = set()
    ordered = []
    for scr in reversed(screens):
        for line in scr:
            if line in seen:
                continue
            seen.add(line)
            ordered.append(line)

    out = os.path.abspath(args.out)
    os.makedirs(os.path.dirname(out) or '.', exist_ok=True)
    with open(out, 'w', encoding='utf-8') as f:
        f.write('\n'.join(ordered))

    print(f'\n完成！{len(screens)} 屏 → 去重后 {len(ordered)} 行')
    print(f'输出: {out}')
    return 0


if __name__ == '__main__':
    sys.exit(main())
```
