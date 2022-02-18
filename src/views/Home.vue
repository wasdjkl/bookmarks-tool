<template>
  <div class="wrapper">
    <div class="control-panel">
      <button type="button" @click="handleLoadFromHtml">Load from HTML</button>
      <button type="button" @click="handleLoadCSV">Load from CSV</button>
      <button type="button" @click="handleLoadJson">Load from JSON</button>
      <br />
      <button type="button" v-if="!show" @click="show = true">Show</button>
      <button type="button" v-if="show" @click="show = false">Hide</button>
      <br />
      <button type="button" @click="handleExportToCSV">Export to CSV</button>
      <button type="button" @click="handleExportToHtml">Export to HTML</button>
      <button type="button" @click="handleExportToJson">Export to JSON</button>
      <br />
      <button type="button" @click="handleUniqByUrl">Uniq By Url</button>
      <button type="button" @click="handleFilterHTTPAndHTTPS">
        Filter HTTP & HTTPS Link
      </button>
      <button type="button" @click="handleFilterNotHTTPAndHTTPS">
        Filter Special Link
      </button>
      Read File Count: {{ readFileCount }} |Bookmarks Count:
      {{ bookmarks.length }}
      <hr />
    </div>
    <virtual-list
      v-if="show"
      class="list"
      :data-key="'id'"
      :data-sources="bookmarks"
      :data-component="item"
      :estimate-size="50"
    />
  </div>
</template>

<script>
import $ from "jquery";
import _ from "lodash";
import { saveAs } from "file-saver";
import { parse, stringify } from "csv";
import Item from "@/components/Item";

export default {
  name: "Home",
  props: {
    msg: String,
  },
  data() {
    return {
      show: false,
      readFileCount: 0,
      bookmarks: [],
      item: Item,
    };
  },
  methods: {
    handleLoadFromHtml() {
      this.loadFile(this.parseHTML, "text/html");
    },
    handleLoadCSV() {
      this.loadFile(this.parseCSV, "text/csv");
    },
    handleLoadJson(){
      this.loadFile(this.parseJSON,'application/json')
    },
    loadFile(parseFunction, fileType) {
      const fileSelector = document.createElement("input");
      fileSelector.type = "file";
      fileSelector.accept = fileType;
      fileSelector.multiple = true;
      fileSelector.onchange = () => {
        Array.from(fileSelector.files).forEach((file, index) => {
          let reader = new FileReader();
          reader.onload = (e) => {
            const data = e.target.result;
            reader = null;
            parseFunction(data);
            this.readFileCount++;
            console.log(
              `${this.readFileCount} ParseDone: ${file.name} Index:${index}`
            );
          };
          reader.readAsText(file);
        });
      };
      fileSelector.click();
    },
    parseHTML(content) {
      let htmlElement = $.parseHTML(content);
      let linkElementList = $("A", htmlElement);
      Array.from(linkElementList).forEach((element) => {
        const folder = this.getFolderPath(element);
        this.bookmarks.push({
          id: Math.random(),
          url: $(element).attr("href"),
          title: $(element).text(),
          addDate: $(element).attr("ADD_DATE"),
          lastModified: $(element).attr("LAST_MODIFIED"),
          icon: $(element).attr("ICON"),
          tags: $(element).attr("TAGS"),
          description: $(element).attr("DESCRIPTION"),
          dataCover: $(element).attr("DATA-COVER"),
          folder,
        });
        element = null;
      });
      linkElementList = null;
      htmlElement = null;
    },
    parseCSV(content) {
      parse(
        content,
        {
          delimiter: ",",
          columns: true,
        },
        (err, output) => {
          console.log(err, output);
          output.forEach((element) => {
            this.bookmarks.push({
              id: Math.random(),
              url: element.url,
              title: element.title,
              addDate: element.created,
              lastModified: element.lastModified,
              icon: element.icon,
              tags: element.tags,
              description: element.description,
              dataCover: element.dataCover,
              folder: element.folder,
            });
          });
        }
      );
    },
    parseJSON(content){
      return content
    },
    getFolderPath(element) {
      const folderList = $(element).parent("DT").parents("DT");
      return Array.from(folderList)
        .map((folder) => {
          return $(folder).children("H3").text();
        })
        .reverse()
        .join("/");
    },
    handleExportToCSV() {
      const csvObject = this.bookmarks.map((value) => {
        return {
          folder: value.folder,
          url: value.url,
          title: value.title,
          description: value.description,
          tags: value.tags,
          created: value.addDate,
          lastModified: value.lastModified,
          icon: value.icon,
          dataCover: value.dataCover,
        };
      });
      stringify(
        csvObject,
        {
          header: true,
          bom: true,
        },
        (err, output) => {
          const blob = new Blob([output], {
            type: "text/csv;charset=utf-8",
          });
          saveAs(blob, `bookmarks_${this.bookmarks.length}.csv`);
        }
      );
    },
    handleExportToHtml() {
      const bookmarks = [];
      this.bookmarks.forEach((item) => {
        const folder = item.folder;
        const path = folder.split("/");
        let current = {
          children: bookmarks,
        };
        if (folder.trim().length <= 0) {
          console.log(item);
          current.children.push(item);
        } else {
          path.forEach((value) => {
            let findResult = current.children.find((e) => {
              return e.title === value && e.isFolder;
            });
            if (!findResult) {
              let index = current.children.length;
              current.children.push({
                title: value,
                isFolder: true,
                children: [],
              });
              current = current.children[index];
            } else {
              current = findResult;
            }
          });
          current.children.push(item);
        }
      });
      console.log(bookmarks);

      let htmlContent = `<!DOCTYPE NETSCAPE-Bookmark-file-1>
      <META HTTP-EQUIV="Content-Type" CONTENT="text/html; charset=UTF-8">
      <TITLE>Bookmarks</TITLE>
      <H1>Bookmarks</H1>
      <DL><p>
      `;

      htmlContent += this.createBookmarkElement({
        title: "Bookmarks",
        isFolder: true,
        children: bookmarks,
      });
      htmlContent += `</DL><p>`;
      const blob = new Blob([htmlContent], {
        type: "text/html;charset=utf-8",
      });
      saveAs(blob, `bookmarks_${this.bookmarks.length}.html`);
    },
    handleExportToJson(){
      const blob = new Blob([JSON.stringify(this.bookmarks)], {
        type: "application/json;charset=utf-8",
      });
      saveAs(blob, `bookmarks_${this.bookmarks.length}.json`);
    },
    createBookmarkElement(bookmarksData) {
      console.log(bookmarksData);
      let htmlContent = "";
      if (bookmarksData.isFolder) {
        const h3Html = document.createElement("H3");
        h3Html.setAttribute("ADD_DATE", "0");
        h3Html.setAttribute("LAST_MODIFIED", "0");
        h3Html.textContent = bookmarksData.title;
        htmlContent += `<DT>${h3Html.outerHTML}\n`;
        htmlContent += `<DL><p>\n`;
        bookmarksData.children.forEach((item) => {
          htmlContent += this.createBookmarkElement(item);
        });
        htmlContent += `</DL><p>\n`;
      } else {
        let link = this.createLinkElement(bookmarksData);
        htmlContent += `<DT>${link.outerHTML}\n`;
      }
      return htmlContent;
    },
    createLinkElement(item) {
      let link = document.createElement("a");
      link.href = item.url;
      link.textContent = item.title;
      link.setAttribute("ADD_DATE", item.addDate);
      link.setAttribute("LAST_MODIFIED", item.lastModified);
      link.setAttribute("ICON", item.icon);
      link.setAttribute("DATA_COVER", item.dataCover);
      link.setAttribute("TAGS", item.tags);
      link.setAttribute("DESCRIPTION", item.description);
      return link;
    },
    handleUniqByUrl() {
      this.bookmarks = _.uniqBy(this.bookmarks, "url");
    },
    handleFilterHTTPAndHTTPS() {
      this.bookmarks = _.filter(this.bookmarks, (item) => {
        return /^(http|https):/.test(item.url);
      });
    },
    handleFilterNotHTTPAndHTTPS() {
      this.bookmarks = _.filter(this.bookmarks, (item) => {
        return !/^(http|https):/.test(item.url);
      });
    },
  },
};
</script>

<style scoped lang="scss">
.wrapper {
  display: flex;
  flex-direction: column;
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}

.list {
  overflow-x: hidden;
  overflow-y: auto;
}
</style>
