<template>
    <div>
        <el-container>
            <el-aside>
                <el-form label-width="80px" :model="formObj">
                    <el-form-item label="用户名">
                        <el-input v-model="formObj.name"></el-input>
                    </el-form-item>
                    <el-form-item label="密码">
                        <el-input v-model="formObj.pw"></el-input>
                    </el-form-item>
                </el-form>
                <el-button @click="startFn" v-if="!isLogin">开始</el-button>
                <el-button @click="toDownLoadFolder">点击进行下载</el-button>
                <el-checkbox-group v-model="checkFiles">
                    <el-row :gutter="10">
                        <el-col v-for="item in errFiles" :key="item.name">
                            <el-checkbox :label="item" border>
                                {{item.name}}
                            </el-checkbox>
                        </el-col>
                    </el-row>
                    <el-button @click="downErrFile">下载出错的文件</el-button>
                </el-checkbox-group>
            </el-aside>
            <el-aside>
                <el-checkbox-group v-model="checkFolders">
                    <el-row :gutter="10">
                        <el-col v-for="item in folders" :key="item.name">
                            <el-checkbox :label="item" border>
                                {{item.name}}
                            </el-checkbox>
                        </el-col>
                    </el-row>
                </el-checkbox-group>
            </el-aside>
            <el-aside>
                <ul>
                    <li><i class="el-icon-warning"></i>注意：在软件运行期间请不要手动关闭自动打开的浏览器！</li>
                    <transition-group name="el-fade-in-linear">
                        <li v-for="item in msgs" :key="item.id">
                            <i :class="item.icon"></i>- {{item.text}}
                        </li>
                    </transition-group>
                </ul>
            </el-aside>
        </el-container>
    </div>
</template>
<script>
const charset = require('superagent-charset'); //解决乱码问题:
const superagent = require('superagent'); //发起请求
charset(superagent);
const async = require('async'); //异步抓取
const url = 'https://jinshuju.net/login';
const fs = require('fs')
const puppeteer = require('puppeteer');
const path = require('path')
let page = null;
let browser = null;
let concurrencyCount = 0; //并发计数器
function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms))
}

// 递归创建目录 同步方法
function mkdirsSync(dirname) {
    if (fs.existsSync(dirname)) {
        return true;
    } else {
        if (mkdirsSync(path.dirname(dirname))) {
            fs.mkdirSync(dirname);
            return true;
        }
    }
}
export default {
    name: "ss",
    data() {
        return {
            isLogin: false,
            checkFiles: [], //出错的文件，记录在这里，可以点击再次下载
            formObj: {
                name: 'test@qq.com',
                pw: 'test'
            },
            msgs: [],
            folders: [],
            checkFolders: [],
            errFiles: [], //用来记录出错的file
        }
    },
    methods: {
        pushMsg(msg, type = 'info') {
            this.msgs.push({
                id: Math.random(),
                text: msg,
                type: type,
                icon: `el-icon-${type}`,
            })
        },
        async startFn() {
            browser = await puppeteer.launch({
                headless: false,
            });
            page = await browser.newPage();
            // 设置浏览器视窗
            await page.setViewport({
                width: 1376,
                height: 768,
            });
            await this.login();
        },
        async login() {
            await page.goto(url);
            await sleep(1000)
            this.pushMsg('准备登陆', 'info')
            const aa = await page.evaluate(() => {
                $userName = document.getElementById('auth_key');
                $pw = document.getElementById('password');
                $userName.value = '1529590866@qq.com'
                $pw.value = 'shichang0987';
            })
            await page.waitForSelector('.submit-field .gd-btn-primary-solid');
            await page.focus('.submit-field .gd-btn-primary-solid')
            await page.keyboard.press('Enter');
            await sleep(4 * 1000)
            this.isLogin = true;
            this.pushMsg('登录成功！', 'success')
            await this.getAllFolders()
        },
        async getAllFolders() {
            try {
                await page.goto('https://jinshuju.net/form_tags/all_folders')
                console.log('准备抓取所有文件夹');
                this.pushMsg('准备抓取所有文件夹', 'info')
                await sleep(1000)
                await page.waitForSelector('#cover_list');
            } catch (e) {
                console.log(e)
                this.pushMsg('在抓取文件夹出错请重试！', 'error')
            }

            try {
                this.folders = await page.$$eval('#cover_list a', link => {
                    return link.map(a => {
                        let name = a.getElementsByClassName('name')[0].innerText
                        console.log(name)
                        return {
                            href: a.href.trim(),
                            name,
                        }
                    });
                });
            } catch (err) {
                console.log(err);
                this.pushMsg('在抓取文件夹出错请重试！', 'error')
            };
            this.pushMsg(`抓取文件夹成功：共计:${this.folders.length}`, 'success')
            this.pushMsg(`请您勾选需要下载的文件夹！然后点击下载！`)
        },
        async toDownLoadFolder() {
            if (this.checkFolders.length === 0) {
                this.$message.error('请选择文件夹！');
            }
            for (let folder of this.checkFolders) {
                let files = await this.getFolderFile(folder);
                concurrencyCount = 0;
                async.mapLimit(files, 7, (file, callback) => {
                    this.downFolderFile(file, callback);
                }, (err, result) => {
                    console.log('爬虫数据完毕~~~~~~~');
                    console.log(result.length)
                    if (result.length === files.length) {
                        this.$message.success(`${folder.name}文件夹下所有文件下载成功！`)
                        this.pushMsg(`${folder.name}文件夹下所有文件下载成功！`, 'success')
                    }

                    // 爬虫结束后的回调，可以做一些统计结果
                    return false;
                });
            }
        },
        // 获取文件夹下的所有文件
        async getFolderFile(foler) {
            if (!foler) return;
            console.log(`打开文件夹：${foler.name}`)
            let dirname = foler.name.replace(/[^\u4e00-\u9fa5a-zA-Z0-9]/ig, '');
            try {
                // 创建目录
                mkdirsSync(dirname);
                // 删除目录下的所有文件
                var dirList = fs.readdirSync('./' + dirname);
                dirList.forEach(function(fileName) {
                    fs.unlinkSync('./' + dirname + '/' + fileName);
                });
            } catch (e) {
                this.pushMsg(`请手动删除掉相关文件夹下面的文件！`)
            }

            this.pushMsg(`获取：${foler.name}下面的文件ing`)
            try {
                const page = await browser.newPage();
                console.log(foler.href)
                await page.goto(foler.href);
                await sleep(2000)
                await page.waitForSelector('#cover_list');
                for (let i = 0; i < 100; i++) {
                    await page.keyboard.press('ArrowDown', {
                        delay: 100
                    })
                }
                let files;
                files = await page.$$eval('#cover_list .form-item-wrapper', link => {
                    return link.map(a => {
                        let href = a.getElementsByTagName('a')[0].href
                        let count = a.getElementsByClassName('count')[0].innerText
                        let name = a.getElementsByClassName('name')[0].innerText
                        if (count - 0 > 0) {
                            return {
                                href,
                                count,
                                name,
                            }
                        }
                    });
                });
                files = files.filter(file => {
                    return file
                })
                this.pushMsg(`获取：${foler.name}下面的文件成功：共计${files.length}`, 'success')
                files = files.map(file => {
                    return Object.assign({
                        dirname: dirname
                    }, file)
                })
                return files;
            } catch (e) {
                console.log(e)
                this.pushMsg(`获取：${foler.name}下面的文件失败！`, 'error')
                this.pushMsg(`请您打开软件重试！`, );
                this.$message.error('请您打开软件重试！')
            }
        },
        // 下载一个文件夹下面的所有文件！
        async downFolderFile(file, callback) {
            if (!file) return;
            concurrencyCount++;
            let fileSrc = await this.getFileSrc(file);
            concurrencyCount--;
            file.fileSrc = fileSrc;
            callback(null, file);
            // 下载
            this.downloadFile(file)

        },
        // 获取一个文件的真正地址
        async getFileSrc(file, callback) {
            if (!file) return;
            let fileSrc;
            if (callback) {
                concurrencyCount++;
            }
            try {
                const page = await browser.newPage()
                    // 设置浏览器视窗
                await page.setViewport({
                    width: 1376,
                    height: 768,
                });
                await page.goto(file.href);
                await sleep(1000)
                await page.waitForSelector('#form_nav');
                await sleep(800)
                const f = await page.$eval('#form_nav>#entries_nav', dom => dom.href)
                await page.goto(f);
                await page.waitForSelector('#tb_entries_grid_toolbar_item_export_icon')
                    // await sleep(2000)
                    // 判断有无下载按钮
                let hasDownBtn = await page.evaluate(() => {
                        let download = document.getElementsByClassName('gd-link-group')
                        if (download && download.length > 0) return true
                        else return false
                    })
                    // 如果有下载按钮，先点击取消，
                if (hasDownBtn) {
                    await page.click('#tb_entries_grid_toolbar_item_export_icon .gd-link-group a[data-role=remove]')
                }
                // 重新生成下载按钮
                await sleep(1100)
                await page.click('#tb_entries_grid_toolbar_item_export_icon .dropdown-toggle')
                await sleep(2000)
                    // 点击导出所有列
                await page.click('.export_all_columns')
                await sleep(2000)

                // 这里要进行判断，看有无弹框
                let hasModal = await page.evaluate(() => {
                    let download = document.getElementById('export_merge_cells_modal')
                    if (download) return true
                    else return false
                })
                console.log(`有无弹框？${hasModal}`)
                if (hasModal) {
                    await page.click('a.submit.gd-btn.gd-btn-primary')
                }
                await sleep(10000)
                    // 得到文件真正地址
                fileSrc = await page.evaluate(() => {
                    let download = document.getElementsByClassName('gd-link-group')
                    if (download.length == 0) return false
                    if (download) {
                        let downhref = download[0].getElementsByTagName('a')[0].href
                        return downhref;
                    } else {
                        return false
                    }
                });
                console.log(`${file.name}--${fileSrc}`)
                page.close()
            } catch (err) {
                this.errFiles.push(file);
                this.pushMsg(`获取${file.name}失败!`)
            }
            if (callback) {
                concurrencyCount--;
                callback(null, ' html content');
            }
            return fileSrc
        },
        // 下载文件
        async downloadFile(file) {
            let name = file.name.replace(/[^\u4e00-\u9fa5a-zA-Z0-9]/ig, '')
            let write = fs.createWriteStream(`./${file.dirname}/${name}.xls`)
            superagent.get(file.fileSrc).pipe(write)
        },
        async downErrFile() {
            if (this.checkFiles.length === 1) {
                this.$message.error('请选择要下载的文件！')
                return;
            }
            concurrencyCount = 0;
            async.mapLimit(this.checkFiles, 2, async(file, callback) => {
                let fileSrc = await this.getFileSrc(file, callback);
                file.fileSrc = fileSrc;
                // 下载
                this.downloadFile(file)
            }, function(err, result) {
                console.log('爬虫数据完毕~~~~~~~');
                this.$message.success('下载完成了！')
                    // 爬虫结束后的回调，可以做一些统计结果
                return false;
            });
        },
    }
}
</script>
<style scoped>
</style>
