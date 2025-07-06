# TG2RSS
TG Channel to RSS

rss订阅失效，可以将频道订阅，转为rss，方便订阅程序使用
以下代码是在https://dash.deno.com/ 中部署，只需要github登录，然后,创建项目，然后粘贴以下代码就可

使用方法：
链接后加 `/rss?channel=nodeseekc`

如果频道链接是`https://t.me/nodeseekc` 那么只需要`nodeseekc`，拼接到`/rss?channel=`后面

```js
import { DOMParser } from "jsr:@b-fuze/deno-dom";
import { serve } from "https://deno.land/std@0.224.0/http/server.ts";

function escapeXML(str: string): string {
  return `<![CDATA[${str.replace(/]]>/g, "]]>")}]]>`;
}

function cleanHTML(html: string): string {
  return html
    .replace(/ class="[^"]*"/g, "")
    .replace(/ data-[^=]+="[^"]*"/g, "")
    .replace(/ style="[^"]*"/g, "")
    .replace(/ onclick="[^"]*"/g, "")
    .trim();
}

function extractTitle(msgElement: any): string {
  const linkElement = Array.from(msgElement.querySelectorAll('a'))
    .find(a => {
      const href = a.getAttribute("href") || "";
      return href && !href.startsWith("https://t.me/");
    });
  if (linkElement) {
    const linkText = linkElement.textContent?.trim();
    if (linkText && linkText.length > 0) {
      return linkText;
    }
  }
  
  const textElement = msgElement.querySelector(".tgme_widget_message_text");
  if (textElement) {
    const text = textElement.textContent || "";
    const firstLine = text.split("\n")[0]?.trim();
    if (firstLine && firstLine.length > 0) {
      return firstLine.length > 100 ? firstLine.substring(0, 100) + "..." : firstLine;
    }
  }
  
  return "No title";
}

function extractDescription(msgElement: any): string {
  const textElement = msgElement.querySelector(".tgme_widget_message_text");
  if (!textElement) return "No content";
  
  const preElement = textElement.querySelector("pre");
  if (preElement) {
    const preContent = preElement.innerHTML || "";
    return cleanHTML(preContent);
  }
  
  const textContent = textElement.textContent || "";
  const lines = textContent.split('\n');
  
  const contentLines = lines.slice(1).filter(line => line.trim().length > 0);
  
  if (contentLines.length > 0) {
    return contentLines.map(line => line.trim()).join('<br>');
  }
  
  return "No content";
}

function extractLink(msgElement: any, channel: string, messageId: string): string {
  const externalLink = Array.from(msgElement.querySelectorAll('a'))
    .find(a => {
      const href = a.getAttribute("href") || "";
      return href && !href.startsWith("https://t.me/");
    });
  
  if (externalLink) {
    const href = externalLink.getAttribute("href");
    if (href) {
      return href;
    }
  }
  
  return `https://t.me/${channel}/${messageId}`;
}

async function fetchChannelMessages(channel: string, limit: number = 10): Promise<any[]> {
  const url = `https://t.me/s/${channel}?embed=1&before=0`;
  const response = await fetch(url, { headers: { "Accept": "text/html; charset=utf-8" } });
  const html = await response.text();
  
  const parser = new DOMParser();
  const doc = parser.parseFromString(html, "text/html");
  if (!doc) throw new Error("Failed to parse HTML");

  const messages = Array.from(doc.querySelectorAll(".tgme_widget_message"))
    .sort((a, b) => {
      const idA = parseInt(a.getAttribute("data-post")?.split("/")[1] || "0");
      const idB = parseInt(b.getAttribute("data-post")?.split("/")[1] || "0");
      return idB - idA;
    })
    .slice(0, limit);

  return messages.map((msg, index) => {
    const messageId = msg.getAttribute("data-post")?.split("/")[1] || `temp-${index}`;
    const title = extractTitle(msg);
    const description = extractDescription(msg);
    const link = extractLink(msg, channel, messageId);
    const dateStr = msg.querySelector(".tgme_widget_message_date time")?.getAttribute("datetime") || new Date().toISOString();
    
    return {
      id: messageId,
      title,
      description,
      link,
      date: new Date(dateStr).getTime() / 1000
    };
  });
}

function generateRSS(messages: any[], channel: string): string {
  const now = new Date().toUTCString();
  let rss = `<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
  <channel>
    <title>Abbai Telegram channel: https://t.me/IonMagic</title>
    <link>https://t.me/${channel}</link>
    <description>Telegram channel RSS @${channel}</description>
    <lastBuildDate>${now}</lastBuildDate>`;

  for (const msg of messages) {
    const date = new Date(msg.date * 1000).toUTCString();
    rss += `
    <item>
      <title>${escapeXML(msg.title)}</title>
      <link>${msg.link}</link>
      <description>${escapeXML(msg.description)}</description>
      <pubDate>${date}</pubDate>
      <guid>${msg.link}</guid>
    </item>`;
  }

  rss += `
  </channel>
</rss>`;
  return rss;
}

async function handleRequest(req: Request): Promise<Response> {
  try {
    const url = new URL(req.url);
    const channel = url.searchParams.get("channel");
    if (!channel) throw new Error("Channel parameter is required (e.g., /rss?channel=linux_do_channel)");
    
    const messages = await fetchChannelMessages(channel, 10);
    const rssContent = generateRSS(messages, channel);
    return new Response(rssContent, {
      headers: { "Content-Type": "application/rss+xml; charset=utf-8" },
    });
  } catch (error) {
    return new Response(`Error: ${error.message}`, { status: 500 });
  }
}

const RSS_PORT = 8000;
console.log(`RSS server running at http://localhost:${RSS_PORT}/rss`);
await serve(handleRequest, { port: RSS_PORT });
```

1. 构建 Docker 镜像：
   ```dockerfile
   FROM denoland/deno:latest
   WORKDIR /app
   COPY . .
   RUN deno cache server.ts
   EXPOSE 8000
   CMD ["deno", "run", "--allow-net", "server.ts"]
   ```

2. 使用 Docker Compose：
   ```yaml
   version: '3'
   services:
     rss:
       build: .
       ports:
         - "8000:8000"
   ```

3. 启动并后台运行：
   ```bash
   docker-compose up -d
   ```
4. 访问 http://服务器IP:8000/rss?channel=xxx
