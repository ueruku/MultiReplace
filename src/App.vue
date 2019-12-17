<template>
  <div id="app" style="width: 80%; margin: 0px auto;">
    <div style="text-align: left; font-size: 2em; color: gray;">MultiReplace</div>
    <div class="input">
      <label>置換表の種類</label>
      <select v-model="textType">
        <option>TSV</option>
        <option>CSV</option>
      </select>
      <hr />
      <input type="checkbox" id="cb_wordDelete" v-model="cb_wordDelete" />
      <label for="cb_wordDelete">空文字に置換可能</label>
      <hr />
    </div>
    <label>置換表 - {{ textType }}</label>
    <br />
    <textarea
      v-model="replacementTable"
      v-on:keydown.prevent.tab="inputTab"
      style="width: 80%; height: 10em;"
      wrap="off"
    ></textarea>
    <br />
    <br />
    <label>テキスト</label>
    <br />
    <textarea class="result" v-model="text" style="width: 80%; height: 10em;" wrap="off"></textarea>
    <br />
    <br />
    <label>置換後テキスト</label>
    <br />
    <textarea class="result" v-model="replacedText" style="width: 80%; height: 10em;" wrap="off"></textarea>

    <div style="width: fit-content; margin: 0px auto; background-color: rgb(204, 204, 204);">
      ※使い方
      <div style="width: fit-content; text-align: left; margin: 0px auto;">
        ・置換表の種類を選ぶ
        <br />・置換前+置換後が乗ってるテキストを`置換表`を貼り付ける
        <br />・置換したいテキストを`テキスト`に貼り付ける
        <br />・`置換後テキスト`に出てくる
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from "vue";

export default Vue.extend({
  name: "app",
  data() {
    return {
      textType: "TSV",
      cb_wordDelete: false,
      replacementTable: "",
      text: ""
    };
  },
  // 算出プロパティ
  computed: {
    /** ボックスのスタイル */
    replacedText(): string {
      let table: string[][] = [];
      if (this.textType === "TSV")
        table = this.combartTsvToTable(this.replacementTable);
      if (this.textType === "CSV")
        table = this.combartCsvToTable(this.replacementTable);

      // 余計な空行を削除
      table = this.deleteBlankRowFromTable(table);
      if (table.length === 0) return "--置換表が未設定です--";

      // 末尾の空白を削除
      table = this.deleteTrailingBlankFromTable(table);

      // １行のカラム数は2つにする(置換前と置換後の2行のみ)
      table = table.map(row => this.resizeArray(row, 2, ""));

      // プレフィックス対策で置換対象の文字列が長いものから順に行うようにする
      table.sort((row1, row2) => (row1[0].length < row2[0].length ? 1 : -1));

      // 置換後の文字列を置換するような事が起きないよう、split joinで置換する
      let replacedText = this.replaceTextByWords(this.text, table);
      if (this.text === replacedText) return "--置換対象の文字列がありません--";

      return replacedText;
    }
  },
  methods: {
    /** 2次元配列から空行を削除 */
    deleteBlankRowFromTable(table: string[][]): string[][] {
      return table.filter(row => row.some(col => col !== ""));
    },

    /** 2次元配列から末尾が空文字の項を削除 */
    deleteTrailingBlankFromTable(table: string[][]): string[][] {
      return table.map(row => {
        const index =
          row.length -
          row
            .slice()
            .reverse()
            .findIndex(col => col !== "");
        if (row.length < index) return [];
        return row.slice(0, index);
      });
    },

    /** CSV→2次元配列 */
    combartCsvToTable(text: string): string[][] {
      let csvRows: string[][] = [];
      let csvRow: string[] = [];

      // 項目の文字列をくみ上げる(一時変数)
      let value = "";

      // 「"」内フラグ
      let isContent = false;

      // 1文字づつ進める
      const textArr =
        text.match(/[\uD800-\uDBFF][\uDC00-\uDFFF]|[\s\S]/g) || [];
      for (let i = 0; i < textArr.length; ++i) {
        let char1 = textArr[i];

        if (isContent) {
          // 「"」内
          if (char1 === '"') {
            // 「"」が来たら2文字目を見る
            let char2 = textArr[i + 1];
            if (char2 === '"') {
              // 「""」ならエスケープされた「"」なので、「"」をセットする
              value += '"';
              // 2文字目は対応をしたので飛ばす
              ++i;
            } else {
              isContent = false;
            }
          } else {
            value += char1;
          }
          continue;
        }

        // 「"」外
        if (char1 === '"') {
          isContent = true;
        } else if (char1 === ",") {
          csvRow.push(value);
          value = "";
        } else if (char1 === "\n") {
          // 改行が来たら改行する
          csvRow.push(value);
          value = "";
          csvRows.push(csvRow);
          csvRow = [];
        } else {
          value += char1;
        }
      }

      if (value !== "") {
        // valueが有るならセット
        csvRow.push(value);
        value = "";
        csvRows.push(csvRow);
        csvRow = [];
      }

      return csvRows;
    },
    /** TSV→2次元配列 */
    combartTsvToTable(tsv: string): string[][] {
      return tsv.split("\n").map(row => row.split("\t"));
    },

    /** 配列の長さを変える
     *
     * - 元の配列より長くする：末尾に初期値を埋めて長くする
     * - 短くする：末尾を切り落とす
     * - 同じサイズ：何もしない
     * lengthにマイナスを入れるとUB。
     */
    resizeArray(arr: any[], length: number, initValue: any): any[] {
      const len = length - arr.length;
      if (len === 0) return arr;
      if (len < 0) return arr.slice(0, length);
      return arr.concat(Array(len).fill(initValue));
    },

    /** タブ入力をさせる */
    inputTab(event: any) {
      let obj = event.target;

      // 現在のカーソルの位置を取得
      const cursorPosition = obj.selectionStart;

      // カーソル位置にタブ挿入
      obj.value =
        obj.value.substr(0, cursorPosition) +
        "\t" +
        obj.value.substr(cursorPosition, obj.value.length);

      // カーソルを進める
      obj.selectionEnd = cursorPosition + 1;
    },

    /** 複数の文字列で置換を行う
     *
     * - 置換後の文字列を置換するような事が起きないよう、split joinで置換する
     * - 空文字置換がダメな時に、空文字置換が来たら、置換前の文字列で代わりに置換する
     */
    replaceTextByWords(
      text: string,
      replacementTable: string[][],
      index: number = 0
    ): string {
      const word = replacementTable[index][0];
      // 空文字置換がダメな時に、空文字置換が来たら、置換前の文字列で代わりに置換する
      const replacedWord =
        !this.cb_wordDelete && replacementTable[index][1] === ""
          ? word
          : replacementTable[index][1];

      // 置換前の文字列でテキストを分割する
      let replacedSplit = text.split(word);

      // 次の置換があるなら、分割された個々のテキストに対して個別に置換を行う(再帰)
      if (index < replacementTable.length - 1) {
        replacedSplit = replacedSplit.map(partText => {
          return this.replaceTextByWords(partText, replacementTable, index + 1);
        });
      }

      // 置換後の文字列で結合して末尾に付与
      return replacedSplit.join(replacedWord);
    }
  }
});
</script>

<style>
#app {
  font-family: "Lato", "Noto Sans JP", "游ゴシック Medium", "游ゴシック体",
    "Yu Gothic Medium", YuGothic, "ヒラギノ角ゴ ProN",
    "Hiragino Kaku Gothic ProN", "メイリオ", Meiryo, "ＭＳ Ｐゴシック",
    "MS PGothic", sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
