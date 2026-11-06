"""消息推送：当前实现 Server 酱（sct.ftqq.com）"""
import json
import logging
import urllib.parse
import urllib.request
import os
import re
from typing import Optional


class ServerChanNotifier:
    """Server 酱·Turbo 版：https://sct.ftqq.com/

    免费额度：每日 5 条；超出付费。
    """

    ENDPOINT_TPL = "https://sctapi.ftqq.com/{key}.send"

    def __init__(self, send_key: str, logger: logging.Logger,
                 channel: Optional[str] = None):
        self.send_key = send_key.strip()
        self.logger = logger
        self.channel = channel  # 可指定推送通道，不填默认

    def send(self, title: str, desp: str = "") -> bool:
        if not self.send_key:
            self.logger.debug("Server酱 SendKey 未配置，跳过推送")
            return False
        url = self.ENDPOINT_TPL.format(key=self.send_key)
        payload = {
            # 限60字符
            "title": title[:60],
            # 支持 markdown
            "desp": desp[:32000],
        }
        if self.channel:
            payload["channel"] = self.channel

        data = urllib.parse.urlencode(payload).encode("utf-8")
        req = urllib.request.Request(
            url, data=data, method="POST",
            headers={"Content-Type": "application/x-www-form-urlencoded"},
        )
        try:
            with urllib.request.urlopen(req, timeout=10) as resp:
                body = resp.read().decode("utf-8", "ignore")
                obj = json.loads(body)
                code = obj.get("code", -1)
                if code == 0:
                    self.logger.info("Server酱 推送成功: %s", title)
                    return True
                self.logger.warning("Server酱 推送失败 code=%s body=%s",
                                    code, body[:200])
                return False
        except Exception as e:
            self.logger.warning("Server酱 推送异常: %s", e)
            return False


class BarkNotifier:
    """使用 Bark 向 iPhone 发送机票提醒"""

    def __init__(self, key: str, logger: logging.Logger):
        self.key = key.strip()
        self.logger = logger

    def send(self, title: str, desp: str = "") -> bool:
        # 提取机票查询链接
        link = re.search(
            r"\[[^\]]+\]\((https?://[^)]+)\)", desp
        )

        # 将原来的 Markdown 内容转成普通文本
        body = re.sub(
            r"\[([^\]]+)\]\(https?://[^)]+\)",
            r"\1",
            desp
        )
        body = re.sub(
            r"^##\s+", "", body, flags=re.MULTILINE
        ).replace("**", "")

        payload = {
            "title": title,
            "body": body,
            "group": "机票监控",
        }

        if link:
            payload["url"] = link.group(1)

        request = urllib.request.Request(
            f"https://api.day.app/{self.key}",
            data=json.dumps(payload, ensure_ascii=False).encode("utf-8"),
            headers={"Content-Type": "application/json; charset=utf-8"},
            method="POST",
        )

        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                result = json.loads(response.read().decode("utf-8"))
                if result.get("code") == 200:
                    self.logger.info("Bark 推送成功")
                    return True
                self.logger.warning(
                    "Bark 推送失败，code=%s", result.get("code")
                )
        except Exception as e:
            self.logger.warning(
                "Bark 推送异常：%s", type(e).__name__
            )

        return False



def build_notifier(cfg: dict, logger: logging.Logger):
    # 优先使用 Bark
    bark_key = os.environ.get("BARK_KEY", "").strip()

    if bark_key:
        return BarkNotifier(bark_key, logger)

    # 没有 Bark Key 时继续使用原有 Server酱
    if not cfg:
        return None

    sc = cfg.get("serverchan") or {}
    if sc.get("enabled") and sc.get("send_key"):
        return ServerChanNotifier(
            send_key=sc["send_key"],
            logger=logger,
            channel=sc.get("channel"),
        )

    return None

