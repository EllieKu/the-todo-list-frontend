<template>
  <div>
    <el-button type="primary" @click="openCreateDialog" style="margin-bottom: 20px;">
      新增 Todo
    </el-button>
    
    <el-table ref="tableRef" row-key="id" :data="tableData" style="width: 100%" @sort-change="handleSortChange">
    <el-table-column
      prop="is_done"
      label="Done"
      width="100"
    >
      <template #default="scope">
        <el-checkbox
          :model-value="scope.row.is_done === 1"
          @change="updateTodoStatus(scope.row.id, $event)"
        />
      </template>
    </el-table-column>
    <el-table-column prop="task" label="Task" :formatter="formatter" sortable="custom" />   
    <el-table-column
      prop="deadline"
      label="Deadline"
      width="180"
      sortable="custom"
      :formatter="deadlineFormatter"
    />
    <el-table-column
      prop="priority"
      label="Priority"
      width="100"
      sortable="custom"
    >
      <template #default="scope">
        <el-tag
          :type="scope.row.priority === 0 ? 'info' : scope.row.priority === 1 ? 'warning' : 'danger'"
          disable-transitions
          >{{ scope.row.priority === 0 ? 'Low' : scope.row.priority === 1 ? 'Medium' : 'High' }}</el-tag
        >
      </template>
    </el-table-column>

    <el-table-column
      label="操作"
      width="150"
    >
      <template #default="scope">
        <el-button
          type="primary"
          size="small"
          @click="openEditDialog(scope.row)"
          style="margin-right: 8px;"
        >
          編輯
        </el-button>
        <el-button
          type="danger"
          size="small"
          @click="confirmDelete(scope.row.id)"
        >
          刪除
        </el-button>
      </template>
    </el-table-column>
  </el-table>
  
  <el-pagination
    v-model:current-page="currentPage"
    :page-size="pageSize"
    :total="total"
    layout="total, prev, pager, next, jumper"
    @current-change="handleCurrentChange"
    style="margin-top: 20px; justify-content: center;"
  />

  <!-- 編輯 Todo 對話框 -->
  <el-dialog v-model="showEditDialog" title="編輯 Todo" width="500px">
    <el-form :model="editTodoForm" label-width="80px">
      <el-form-item label="任務">
        <el-input v-model="editTodoForm.task" placeholder="請輸入任務內容" />
      </el-form-item>
      <el-form-item label="截止日期">
        <el-date-picker
          v-model="editTodoForm.deadline"
          type="date"
          placeholder="選擇截止日期"
          format="YYYY-MM-DD"
          value-format="YYYY-MM-DD"
        />
      </el-form-item>
      <el-form-item label="優先級">
        <el-select v-model="editTodoForm.priority" placeholder="選擇優先級">
          <el-option label="Low" :value="0" />
          <el-option label="Medium" :value="1" />
          <el-option label="High" :value="2" />
        </el-select>
      </el-form-item>
    </el-form>
    <template #footer>
      <el-button @click="showEditDialog = false">取消</el-button>
      <el-button type="primary" @click="updateTodo">確定</el-button>
    </template>
  </el-dialog>

  <!-- 新增 Todo 對話框 -->
  <el-dialog v-model="showCreateDialog" title="新增 Todo" width="500px">
    <el-form :model="newTodoForm" label-width="80px">
      <el-form-item label="任務">
        <el-input v-model="newTodoForm.task" placeholder="請輸入任務內容" />
      </el-form-item>
      <el-form-item label="截止日期">
        <el-date-picker
          v-model="newTodoForm.deadline"
          type="date"
          placeholder="選擇截止日期"
          format="YYYY-MM-DD"
          value-format="YYYY-MM-DD"
        />
      </el-form-item>
      <el-form-item label="優先級">
        <el-select v-model="newTodoForm.priority" placeholder="選擇優先級">
          <el-option label="Low" :value="0" />
          <el-option label="Medium" :value="1" />
          <el-option label="High" :value="2" />
        </el-select>
      </el-form-item>
    </el-form>
    <template #footer>
      <el-button @click="showCreateDialog = false">取消</el-button>
      <el-button type="primary" @click="createTodo">確定</el-button>
    </template>
  </el-dialog>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'
import { ElMessageBox } from 'element-plus'

import type { TableColumnCtx, TableInstance } from 'element-plus'

const USER_ID = "23f5e96e-24cd-4b58-82a9-3e10b3725b1f"

interface Todo {
  id: number
  task: string
  deadline: string
  priority: number
  is_done: number
}

const tableRef = ref<TableInstance>()
const tableData = ref<Todo[]>([])
const currentPage = ref(1)
const pageSize = ref(5)
const total = ref(0)
const currentSortBy = ref<string>()
const currentSortOrder = ref<'asc' | 'desc'>()
const showCreateDialog = ref(false)
const showEditDialog = ref(false)
const newTodoForm = ref({
  task: '',
  deadline: '',
  priority: 0
})
const editTodoForm = ref({
  id: 0,
  task: '',
  deadline: '',
  priority: 0
})

const fetchTodos = async (sortBy?: string, sortOrder?: 'asc' | 'desc') => {
  try {

    if (sortBy !== undefined) currentSortBy.value = sortBy
    if (sortOrder !== undefined) currentSortOrder.value = sortOrder
    
    const params: any = {
      page_index: currentPage.value - 1,
    }
    
    if (currentSortBy.value) {
      params.sort_by = currentSortBy.value
    }
    
    if (currentSortOrder.value) {
      params.sort_order = currentSortOrder.value
    }
    
    const response = await axios.get('https://the-todo-list-backend.onrender.com/todo', {
      headers: {'user-id': USER_ID},
      params
    })
    tableData.value = response.data.todos
    total.value = response.data.todos_total
  } catch (error) {
    console.error('Failed to fetch todos:', error)
  }
}

const updateTodoStatus = async (id, isDone: boolean) => {
  try {
    await axios.put(
      `https://the-todo-list-backend.onrender.com/todo/${id}`, 
      { is_done: isDone ? 1 : 0 },
      { 
        headers: {
          'user-id': USER_ID,
          'Content-Type': 'application/json'
        }
      }
    )
    const todo = tableData.value.find((t: Todo) => t.id === id)
    if (todo) {
      todo.is_done = isDone ? 1 : 0
    }
  } catch (error) {
    console.error('Failed to update todo status:', error)
    console.error(error.response?.data)
  }
}


const handleCurrentChange = (val: number) => {
  currentPage.value = val
  fetchTodos()
}

const handleSortChange = (sort: any) => {
  if (sort.prop === 'deadline' || sort.prop === 'priority' || sort.prop === 'task') {
    const order = sort.order === 'ascending' ? 'asc' : sort.order === 'descending' ? 'desc' : undefined
    if (order) {
      fetchTodos(sort.prop, order)
    } else {
      // 清除排序
      currentSortBy.value = undefined
      currentSortOrder.value = undefined
      fetchTodos()
    }
  }
}

const createTodo = async () => {
  try {
    // 轉換 deadline 格式為 yyyymmdd
    const deadline = newTodoForm.value.deadline ? newTodoForm.value.deadline.replace(/-/g, '') : ''
    
    await axios.post('https://the-todo-list-backend.onrender.com/todo', 
      {
        task: newTodoForm.value.task,
        deadline: deadline,
        priority: newTodoForm.value.priority,
        is_done: 0
      },
      {
        headers: {
          'user-id': USER_ID,
          'Content-Type': 'application/json'
        }
      }
    )
    showCreateDialog.value = false
    newTodoForm.value = { task: '', deadline: '', priority: 0 }
    fetchTodos()
  } catch (error) {
    console.error('Failed to create todo:', error)
  }
}

const openCreateDialog = () => {
  showCreateDialog.value = true
}

const openEditDialog = (todo: Todo) => {
  editTodoForm.value = {
    id: todo.id,
    task: todo.task,
    deadline: todo.deadline ? `${todo.deadline.substring(0, 4)}-${todo.deadline.substring(4, 6)}-${todo.deadline.substring(6, 8)}` : '',
    priority: todo.priority
  }
  showEditDialog.value = true
}

const updateTodo = async () => {
  try {
    const deadline = editTodoForm.value.deadline ? editTodoForm.value.deadline.replace(/-/g, '') : ''
    
    await axios.put(`https://the-todo-list-backend.onrender.com/todo/${editTodoForm.value.id}`, 
      {
        task: editTodoForm.value.task,
        deadline: deadline,
        priority: editTodoForm.value.priority
      },
      {
        headers: {
          'user-id': USER_ID,
          'Content-Type': 'application/json'
        }
      }
    )
    showEditDialog.value = false
    fetchTodos()
  } catch (error) {
    console.error('Failed to update todo:', error)
  }
}

const deleteTodo = async (id: number) => {
  try {
    await axios.delete(`https://the-todo-list-backend.onrender.com/todo/${id}`, {
      headers: {
        'user-id': USER_ID
      }
    })
    fetchTodos()
  } catch (error) {
    console.error('Failed to delete todo:', error)
  }
}

const confirmDelete = (id: number) => {
  ElMessageBox.confirm('確定要刪除這個 Todo 嗎？', '確認刪除', {
    confirmButtonText: '確定',
    cancelButtonText: '取消',
    type: 'warning'
  }).then(() => {
    deleteTodo(id)
  }).catch(() => {
    // 用戶取消刪除
  })
}


onMounted(() => {
  fetchTodos()
})


const formatter = (row: Todo, column: TableColumnCtx<Todo>) => {
  return row.task
}

const deadlineFormatter = (row: Todo, column: TableColumnCtx<Todo>) => {
  const deadline = row.deadline
  if (deadline && deadline.length === 8) {
    // 將 yyyymmdd 轉換為 yyyy-mm-dd
    return `${deadline.substring(0, 4)}-${deadline.substring(4, 6)}-${deadline.substring(6, 8)}`
  }
  return deadline
}

</script>