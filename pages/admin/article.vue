<template>
  <div>
    <v-container>
      <v-card>
        <v-card-title>
          <v-subheader>文章列表</v-subheader>
          <v-spacer></v-spacer>
          <v-select :items="category_schema" label="选择分类" item-text="name" item-value="value"
            v-model="search"></v-select>
          <v-btn icon @click="search = ''" v-if="search != ''">
            <v-icon>close</v-icon>
          </v-btn>
        </v-card-title>
        <v-data-table :items="posts.edges" :headers="headers" :rows-per-page-text="'每页显示行数'"
          :rows-per-page-items="rows_per_page" :no-data-text="'暂时没有文章'" :pagination.sync="pagination"
          :total-items="posts.aggregate.count" :loading="$apollo.queries.posts.loading"
          :search="search">
          <template slot="items" slot-scope="props">
            <tr @click="readArticle(props.item.node)">
              <td>{{ props.item.node.title }}</td>
              <td>
                {{ convert_category(props.item.node.category) }}
              </td>
              <td>{{ props.item.node.author }}</td>
              <td class="justify-center layout px-0" v-if="!isMobile">
                <v-icon small class="mr-2" @click="editArticle(props.item.node)">
                  edit
                </v-icon>
                <v-icon small @click="deleteArticle(props.item.node)">
                  delete
                </v-icon>
              </td>
            </tr>
          </template>
        </v-data-table>
      </v-card>
      <v-layout justify-end class="mt-2">
        <v-btn open-on-hover v-if="!isMobile" color="primary" @click="createArticle">新增文章</v-btn>
      </v-layout>
    </v-container>
    <!-- PC端编辑文章抽屉 -->
    <v-navigation-drawer :value="editing.status" stateless :hide-overlay="false" right
      fixed temporary :width="700" v-if="!isMobile" id="dwr">
      <v-toolbar flat>
        <v-list>
          <v-list-tile>
            <v-list-tile-title class="title">
              编辑文章
            </v-list-tile-title>
          </v-list-tile>
        </v-list>
      </v-toolbar>
      <v-divider></v-divider>
      <v-layout wrap class="px-3 py-3">
        <v-flex xs12>
          <!-- meta data -->
          <v-form v-model="post_rule.valid">
            <v-text-field v-model="editing.targetItem.title" :rules="post_rule.title" label="标题"></v-text-field>
            <v-text-field v-model="editing.targetItem.author" :rules="post_rule.author" label="作者"></v-text-field>
            <v-select :items="category_schema" label="分类" item-text="name" item-value="value"
              v-model="editing.targetItem.category"></v-select>
          </v-form>
        </v-flex>
        <v-flex xs12>
          <!-- content -->
          <VueEditor v-model="editing.targetItem.content" />
        </v-flex>
        <v-flex xs4 offset-xs8>
          <v-btn flat color="blue" @click="saveEdit" :loading="save_progress || $apollo.queries.post_content.loading"
            :disabled="!post_rule.valid">保存</v-btn>
          <v-btn flat color="error" @click="cancelEdit">取消</v-btn>
        </v-flex>
      </v-layout>
    </v-navigation-drawer>
    <!-- 手机端阅读文章抽屉 -->
    <v-navigation-drawer :value="reading.status" stateless :hide-overlay="false" right
      fixed temporary :width="700" v-if="isMobile">
      <v-toolbar flat>
        <v-list>
          <v-list-tile>
            <v-list-tile-title class="title">
              阅读文章
            </v-list-tile-title>
          </v-list-tile>
        </v-list>
      </v-toolbar>
      <v-divider></v-divider>
      <v-layout wrap class="px-3 py-3">
        <v-flex xs12>
          <!-- meta data -->
          <h2 class="text-xs-center">{{reading.title}}</h2>
          <h3 class="text-xs-center">作者：{{reading.author}}</h3>
        </v-flex>
        <div v-html="reading.content"></div>
        <v-flex xs4 offset-xs8>
          <v-btn color="primary" @click="reading.status = false">返回</v-btn>
        </v-flex>
      </v-layout>
    </v-navigation-drawer>
  </div>
</template>

<script>
import { mapState } from "vuex";
import VueEditor from "@/components/vue-editor";
import gql from "graphql-tag";
export default {
  components: {
    VueEditor
  },
  data() {
    return {
      category_schema: [
        { name: "政策文章", value: "POLICY" },
        { name: "热点新闻", value: "NEWS" },
        { name: "通用农业技术", value: "AGRICULTURAL_TECH" }
      ],
      editing: {
        status: false,
        method: "edit",
        targetItem: {
          title: "",
          content: "",
          author: "",
          category: ""
        },
        backUpItem: {}
      },
      reading: {
        status: false
      },
      headers: [
        {
          text: "文章标题",
          value: "title"
        },
        {
          text: "分类",
          value: "category"
        },
        {
          text: "作者",
          value: "author"
        }
      ],
      posts: {
        aggregate: {
          count: 0
        },
        edges: [
          {
            node: {
              id: "",
              title: "",
              category: "",
              author: "",
              content: ""
            }
          }
        ]
      },
      post_content: "",
      post_id: "",
      post_rule: {
        title: [v => !!v || "需要标题"],
        author: [v => !!v || "需要作者"],
        valid: false
      },
      save_progress: false,
      pagination: {},
      rows_per_page: [20, 50, 100],
      search: ""
    };
  },
  methods: {
    createArticle() {
      this.editing.targetItem.title = "";
      this.editing.targetItem.author = "";
      this.editing.targetItem.category = "";
      this.editing.targetItem.content = "";
      this.editing.backUpItem = null;
      this.editing.status = true;
    },
    editArticle(item) {
      this.editing.status = true;
      document.getElementById("dwr").scrollTop = 0;
      this.$apollo.queries.post_content
        .refetch({
          where: {
            id: item.id
          }
        })
        .then(({ data }) => {
          this.editArticle2(item, data.post_content.edges[0].node.content);
        })
        .catch(err => {
          console.log(err);
        });
    },
    editArticle2(item, content) {
      this.editing.targetItem.title = item.title;
      this.editing.targetItem.author = item.author;
      this.editing.targetItem.category = item.category;
      this.editing.targetItem.content = content;
      this.editing.backUpItem = item;
      this.editing.status = true;
    },
    deleteArticle(item) {
      if (!confirm(`确认删除《${item.title}》这篇文章？`)) {
        return;
      }
      this.$apollo
        .mutate({
          mutation: gql`
            mutation DeletePost($where: PostWhereUniqueInput!) {
              deletePost(where: $where) {
                id
              }
            }
          `,
          variables: {
            where: {
              id: item.id
            }
          }
        })
        .then(data => {
          this.fetchData();
        })
        .catch(err => {
          alert("删除失败");
        });
    },
    readArticle(item) {
      if (this.isMobile) {
        this.reading.title = item.title;
        this.reading.author = item.author;
        this.reading.content = item.content;
        this.reading.status = true;
      }
    },
    saveEdit() {
      if (!this.editing.backUpItem) {
        //create post
        this.save_progress = true;
        this.$apollo
          .mutate({
            mutation: gql`
              mutation CreatePost($newPost: PostCreateInput!) {
                createPost(data: $newPost) {
                  id
                }
              }
            `,
            variables: {
              newPost: {
                title: this.editing.targetItem.title,
                category: this.editing.targetItem.category,
                author: this.editing.targetItem.author,
                content: this.editing.targetItem.content
              }
            }
          })
          .then(data => {
            console.log(data);
            this.fetchData();
            this.save_progress = false;
            this.editing.status = false;
          })
          .catch(err => {
            console.log(err);
            this.save_progress = false;
          });
      } else {
        //update post
        this.save_progress = true;
        this.$apollo
          .mutate({
            mutation: gql`
              mutation UpdatePost(
                $data: PostUpdateInput!
                $where: PostWhereUniqueInput!
              ) {
                updatePost(data: $data, where: $where) {
                  id
                }
              }
            `,
            variables: {
              data: {
                title: this.editing.targetItem.title,
                category: this.editing.targetItem.category,
                author: this.editing.targetItem.author,
                content: this.editing.targetItem.content
              },
              where: {
                id: this.editing.backUpItem.id
              }
            }
          })
          .then(data => {
            this.fetchData();
            this.save_progress = false;
            this.editing.status = false;
            console.log(data);
          })
          .catch(err => {
            this.save_progress = false;
            console.log(err);
            alert("保存出错！");
          });
      }
    },
    cancelEdit() {
      if (confirm("将丢弃所有更改，确实取消？")) {
        this.editing.status = false;
      }
    },
    fetchData() {
      let myOrder = null;
      if (this.pagination.sortBy == "title") {
        myOrder = this.pagination.descending ? "title_DESC" : "title_ASC";
      } else if (this.pagination.sortBy == "category") {
        myOrder = this.pagination.descending ? "category_DESC" : "category_ASC";
      } else if (this.pagination.sortBy == "author") {
        myOrder = this.pagination.descending ? "author_DESC" : "author_ASC";
      }
      this.$apollo.queries.posts.refetch({
        first: this.pagination.rowsPerPage || 20,
        skip: (this.pagination.page - 1) * this.pagination.rowsPerPage || 0,
        order: myOrder,
        where:
          this.search == ""
            ? null
            : {
                category: this.search
              }
      });
    },
    convert_category(item) {
      let result = this.category_schema.find(({ value }) => {
        return value == item;
      });
      if (result) {
        return result.name;
      } else {
        console.log(item);
      }
    }
  },
  watch: {
    pagination() {
      console.log(this.pagination);
      this.fetchData();
      this.pagination.totalItems = 10;
    }
  },
  apollo: {
    posts: {
      query: gql`
        query ListPosts(
          $first: Int!
          $skip: Int!
          $order: PostOrderByInput
          $where: PostWhereInput
        ) {
          posts(skip: $skip, first: $first, orderBy: $order, where: $where) {
            aggregate {
              count
            }
            edges {
              node {
                id
                title
                category
                author
              }
            }
          }
        }
      `,
      //reactive params
      variables() {
        return {
          first: 20,
          skip: 0
        };
      }
    },
    post_content: {
      query: gql`
        query GetConent($where: PostWhereInput!) {
          post_content: posts(where: $where) {
            edges {
              node {
                content
              }
            }
          }
        }
      `,
      variables() {
        return {
          where: {
            id: this.post_id || ""
          }
        };
      }
    }
  },
  computed: {
    ...mapState(["isMobile"])
  }
};
</script>

<style scoped>
</style>

