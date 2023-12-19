
## 从 MMD 按帧导出
```batch
python mmd-export-by-frame.py <start> <end>
: <start> 起始帧
: <end>   结束帧
deno run --allow-read=. --allow-write=. mmd-export-by-frame.js <ymmp>
: <ymmp> ymmp 文件名（不含.ymmp）
: 对要复制的图片物件设置备注 ::mmd-export-by-frame:<start>:<end>
```

### `mmd-export-by-frame.py`
```python
# -*- coding: utf-8 -*-
import sys
from time import sleep
from pywinauto.application import Application

start = int(sys.argv[1])
end = int(sys.argv[2])
title_re = r'^MikuMikuDance '

app = Application().connect(title_re = title_re)
MMD = app.window(title_re = title_re)
MMD.draw_outline(colour = 'red')
帧 = MMD.child_window(class_name = 'Edit', found_index = 0)

渲染图像 = app.window(title = '渲染图像')
文件名 = 渲染图像.child_window(class_name = 'Edit', found_index = 0)
保存 = 渲染图像.child_window(title = '保存(&S)', class_name = 'Button')

for i in range(start, end + 1):
  帧.set_edit_text('%d' % (i,))
  帧.type_keys('{VK_RETURN}')
  #sleep(0.2)
  MMD.menu_select('文件(&F)->导出图片文件(&B)')
  文件名.set_edit_text('%d.png' % (i,))
  保存.click()
  MMD.wait('active',timeout = 20)
```


### `mmd-export-by-frame.js`
```javascript
import { dirname, resolve } from 'node:path'

const re = /^::mmd-export-by-frame:(\d+):(\d+)/
const getImageItem = (ymmp) => {
  for (const item of ymmp['Timeline']['Items']) {
    if (item['$type'] === 'YukkuriMovieMaker.Project.Items.ImageItem, YukkuriMovieMaker') {
      if (re.test(item['Remark'])) {
        const [_, start, end] = item['Remark'].match(re)
        item['Remark'] = ''
        return {
          imageItem: JSON.stringify(item),
          start: +start, end: +end,
          startFrame: item['Frame'],
          step: item['Length'],
          filePath: item['FilePath']
        }
      }
    }
  }
  throw new TypeError('Remark: "::mmd-export-by-frame": NotFound')
}

export const main = async (args) => {
  const ymmpText = await Deno.readTextFile(`./${args[0]}.ymmp`)
  const ymmp = JSON.parse(ymmpText)
  const { imageItem, start, end, startFrame, step } = getImageItem(ymmp)
  const items = ymmp['Timeline']['Items']
  let i, frame = startFrame
  for (i = start; i <= end; i++) {
    const item = JSON.parse(imageItem)
    item['Frame'] = frame += step
    item['FilePath'] = resolve(dirname(item['FilePath']), `${i}.png`)
    items.push(item)
  }
  await Deno.writeTextFile(`./${args[0]}_export.ymmp`, JSON.stringify(ymmp))
}
if (import.meta.main) {
  main(Deno.args)
}
```