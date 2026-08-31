# Angular 8 借書系統：書本維護參考

這個功能可以：

1. 查詢書本清單。
2. 選擇一本書並查詢明細。
3. 修改書名、作者與庫存數量。
4. 將修改後的資料送到後端儲存。

同一個功能會分別使用：

- Template-driven Form
- Reactive Form

## 檔案結構

```
src/app/
├─ app.module.ts
├─ app-routing.module.ts
└─ book-maintain/
   ├─ model/
   │  └─ book.interface.ts
   ├─ book.service.ts
   ├─ book-maintain.module.ts
   ├─ book-maintain-routing.module.ts
   ├─ template-book/
   │  ├─ template-book.component.ts
   │  └─ template-book.component.html
   └─ reactive-book/
      ├─ reactive-book.component.ts
      └─ reactive-book.component.html
```

---

# 1. 資料型別

檔案：`book-maintain/model/book.interface.ts`

```tsx
// 書本清單、查詢明細與儲存共用這個型別。
export interface Book {
  bookId: number;
  bookName: string;
  author: string;
  stock: number;
}
```

---

# 2. 呼叫後端 API

檔案：`book-maintain/book.service.ts`

```tsx
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';

import { Book } from './model/book.interface';

@Injectable({
  providedIn: 'root'
})
export class BookService {
  // 為了讓範例簡單，直接寫 API 的共同前綴。
  private readonly bookApi = '/api/book';

  constructor(private http: HttpClient) {}

  // 查詢書本清單，供下拉選單使用。
  // POST /api/book/queryBookList
  queryBookList(): Observable<Book[]> {
    return this.http.post<Book[]>(
      this.bookApi + '/queryBookList',
      null
    );
  }

  // 依照書本 ID 查詢完整資料。
  // POST /api/book/query
  query(bookId: number): Observable<Book> {
    return this.http.post<Book>(
      this.bookApi + '/query',
      bookId
    );
  }

  // 將修改後的完整書本資料送到後端。
  // POST /api/book/update
  save(book: Book): Observable<any> {
    return this.http.post(
      this.bookApi + '/update',
      book
    );
  }
}
```

API 對照：

| Service 方法 | API | Request body | Response |
| --- | --- | --- | --- |
| `queryBookList()` | `/queryBookList` | `null` | `Book[]` |
| `query(bookId)` | `/query` | 書本 ID | `Book` |
| `save(book)` | `/update` | 完整書本資料 | 儲存結果 |

---

# 3. 匯入 HttpClientModule

檔案：`app.module.ts`

```tsx
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';
import { AppRoutingModule } from './app-routing.module';

@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule,

    // Service 要使用 HttpClient，
    // 根 Module 必須匯入這個模組。
    HttpClientModule,

    AppRoutingModule
  ],
  bootstrap: [
    AppComponent
  ]
})
export class AppModule {}
```

---

# 4. Routing 與功能 Module

## 4.1 功能 Routing

檔案：`book-maintain/book-maintain-routing.module.ts`

```tsx
import { NgModule } from '@angular/core';
import {
  RouterModule,
  Routes
} from '@angular/router';

import {
  TemplateBookComponent
} from './template-book/template-book.component';

import {
  ReactiveBookComponent
} from './reactive-book/reactive-book.component';

const routes: Routes = [
  {
    // URL：/book-maintain/template
    path: 'template',
    component: TemplateBookComponent
  },
  {
    // URL：/book-maintain/reactive
    path: 'reactive',
    component: ReactiveBookComponent
  },
  {
    // 只輸入 /book-maintain 時，
    // 預設進入 template 版本。
    path: '',
    redirectTo: 'template',
    pathMatch: 'full'
  }
];

@NgModule({
  // 子功能 Routing 使用 forChild。
  imports: [
    RouterModule.forChild(routes)
  ],
  exports: [
    RouterModule
  ]
})
export class BookMaintainRoutingModule {}
```

## 4.2 功能 Module

檔案：`book-maintain/book-maintain.module.ts`

```tsx
import { CommonModule } from '@angular/common';
import {
  FormsModule,
  ReactiveFormsModule
} from '@angular/forms';
import { NgModule } from '@angular/core';

import {
  BookMaintainRoutingModule
} from './book-maintain-routing.module';

import {
  TemplateBookComponent
} from './template-book/template-book.component';

import {
  ReactiveBookComponent
} from './reactive-book/reactive-book.component';

@NgModule({
  declarations: [
    TemplateBookComponent,
    ReactiveBookComponent
  ],
  imports: [
    // 提供 *ngIf、*ngFor。
    CommonModule,

    // Template-driven Form 使用。
    FormsModule,

    // Reactive Form 使用。
    ReactiveFormsModule,

    BookMaintainRoutingModule
  ]
})
export class BookMaintainModule {}
```

## 4.3 根 Routing

檔案：`app-routing.module.ts`

```tsx
import { NgModule } from '@angular/core';
import {
  RouterModule,
  Routes
} from '@angular/router';

const routes: Routes = [
  {
    path: 'book-maintain',

    // 進入 /book-maintain 時，
    // 才載入 BookMaintainModule。
    loadChildren: () =>
      import('./book-maintain/book-maintain.module')
        .then(module => module.BookMaintainModule)
  },
  {
    path: '',
    redirectTo: 'book-maintain',
    pathMatch: 'full'
  }
];

@NgModule({
  // 全站根 Routing 使用 forRoot。
  imports: [
    RouterModule.forRoot(routes)
  ],
  exports: [
    RouterModule
  ]
})
export class AppRoutingModule {}
```

完成後可以進入：

```
/book-maintain/template
/book-maintain/reactive
```

---

# 5. Template-driven Form

## 5.1 Component

檔案：`book-maintain/template-book/template-book.component.ts`

```tsx
import {
  Component,
  OnInit
} from '@angular/core';
import { NgForm } from '@angular/forms';

import { Book } from '../model/book.interface';
import { BookService } from '../book.service';

@Component({
  selector: 'app-template-book',
  templateUrl: './template-book.component.html'
})
export class TemplateBookComponent implements OnInit {
  // 書本下拉選單。
  bookList: Book[] = [];

  // 目前選擇的書本 ID。
  selectedBookId: number = null;

  // API 查詢回來的完整書本資料。
  book: Book = null;

  constructor(
    private bookService: BookService
  ) {}

  ngOnInit(): void {
    // 頁面開啟時，先查詢書本清單。
    this.queryBookList();
  }

  private queryBookList(): void {
    this.bookService.queryBookList().subscribe(
      bookList => {
        this.bookList = bookList;
      },
      error => {
        console.error(error);
        alert('書本清單查詢失敗');
      }
    );
  }

  onBookChange(): void {
    if (this.selectedBookId === null) {
      this.book = null;
      return;
    }

    // 將選到的書本 ID 傳給後端查詢明細。
    this.bookService.query(
      this.selectedBookId
    ).subscribe(
      book => {
        // Template-driven Form 直接綁定 book。
        this.book = book;
      },
      error => {
        console.error(error);
        alert('書本資料查詢失敗');
      }
    );
  }

  save(form: NgForm): void {
    if (form.invalid) {
      // 顯示所有必填欄位的錯誤。
      form.form.markAllAsTouched();
      return;
    }

    // ngModel 已經將畫面輸入同步到 book。
    this.bookService.save(this.book).subscribe(
      () => {
        // 儲存後將表單改回未修改狀態。
        form.form.markAsPristine();
        alert('儲存成功');
      },
      error => {
        console.error(error);
        alert('儲存失敗');
      }
    );
  }
}
```

## 5.2 HTML

檔案：`book-maintain/template-book/template-book.component.html`

```html
<h1>書本維護：Template-driven Form</h1>

<!-- 書本選擇放在編輯表單外面。 -->
<label>選擇書本</label>

<select
  [(ngModel)]="selectedBookId"
  (ngModelChange)="onBookChange()">

  <option [ngValue]="null">
    請選擇書本
  </option>

  <option
    *ngFor="let item of bookList"
    [ngValue]="item.bookId">
    {{ item.bookName }}
  </option>
</select>

<!-- 查詢到書本後才顯示編輯表單。 -->
<form
  *ngIf="book"
  #bookForm="ngForm"
  (ngSubmit)="save(bookForm)">

  <div>
    <label>書名</label>

    <input
      type="text"
      name="bookName"
      [(ngModel)]="book.bookName"
      required
      #bookName="ngModel">

    <small
      *ngIf="bookName.invalid && bookName.touched">
      請輸入書名
    </small>
  </div>

  <div>
    <label>作者</label>

    <input
      type="text"
      name="author"
      [(ngModel)]="book.author"
      required
      #author="ngModel">

    <small
      *ngIf="author.invalid && author.touched">
      請輸入作者
    </small>
  </div>

  <div>
    <label>庫存數量</label>

    <input
      type="number"
      name="stock"
      [(ngModel)]="book.stock"
      required
      min="0"
      #stock="ngModel">

    <small
      *ngIf="stock.invalid && stock.touched">
      庫存數量不可小於 0
    </small>
  </div>

  <button
    type="submit"
    [disabled]="bookForm.invalid">
    儲存
  </button>
</form>
```

資料流：

```
queryBookList()
  → 選擇書本
  → query(bookId)
  → book
  → [(ngModel)] 修改 book
  → save(book)
```

---

# 6. Reactive Form

## 6.1 Component

檔案：`book-maintain/reactive-book/reactive-book.component.ts`

```tsx
import {
  Component,
  OnInit
} from '@angular/core';

import {
  FormBuilder,
  FormGroup,
  Validators
} from '@angular/forms';

import { Book } from '../model/book.interface';
import { BookService } from '../book.service';

@Component({
  selector: 'app-reactive-book',
  templateUrl: './reactive-book.component.html'
})
export class ReactiveBookComponent implements OnInit {
  bookList: Book[] = [];
  bookForm: FormGroup;

  constructor(
    private fb: FormBuilder,
    private bookService: BookService
  ) {
    // Reactive Form 在 Component
    // 建立欄位與驗證規則。
    this.bookForm = this.fb.group({
      // bookId 不顯示，但儲存時要送回後端。
      bookId: [null],

      bookName: [
        '',
        Validators.required
      ],

      author: [
        '',
        Validators.required
      ],

      stock: [
        0,
        [
          Validators.required,
          Validators.min(0)
        ]
      ]
    });
  }

  ngOnInit(): void {
    // 頁面開啟時，先查詢書本清單。
    this.queryBookList();
  }

  private queryBookList(): void {
    this.bookService.queryBookList().subscribe(
      bookList => {
        this.bookList = bookList;
      },
      error => {
        console.error(error);
        alert('書本清單查詢失敗');
      }
    );
  }

  onBookChange(event: Event): void {
    const select =
      event.target as HTMLSelectElement;

    // HTML select 的 value 是字串，
    // 所以需要轉成 number。
    const bookId = Number(select.value);

    if (!bookId) {
      this.bookForm.reset();
      return;
    }

    this.bookService.query(bookId).subscribe(
      book => {
        // API 欄位名稱與 FormControl 相同，
        // 可以直接使用 patchValue。
        this.bookForm.patchValue(book);

        // API 回填不是使用者修改。
        this.bookForm.markAsPristine();
      },
      error => {
        console.error(error);
        alert('書本資料查詢失敗');
      }
    );
  }

  save(): void {
    if (this.bookForm.invalid) {
      this.bookForm.markAllAsTouched();
      return;
    }

    // form.value 的結構與 Book interface 相同。
    const request: Book =
      this.bookForm.value;

    this.bookService.save(request).subscribe(
      () => {
        this.bookForm.markAsPristine();
        alert('儲存成功');
      },
      error => {
        console.error(error);
        alert('儲存失敗');
      }
    );
  }
}
```

## 6.2 HTML

檔案：`book-maintain/reactive-book/reactive-book.component.html`

```html
<h1>書本維護：Reactive Form</h1>

<label>選擇書本</label>

<select (change)="onBookChange($event)">
  <option value="">
    請選擇書本
  </option>

  <option
    *ngFor="let item of bookList"
    [value]="item.bookId">
    {{ item.bookName }}
  </option>
</select>

<form
  [formGroup]="bookForm"
  (ngSubmit)="save()">

  <div>
    <label>書名</label>

    <input
      type="text"
      formControlName="bookName">

    <small
      *ngIf="bookForm.get('bookName').invalid &&
             bookForm.get('bookName').touched">
      請輸入書名
    </small>
  </div>

  <div>
    <label>作者</label>

    <input
      type="text"
      formControlName="author">

    <small
      *ngIf="bookForm.get('author').invalid &&
             bookForm.get('author').touched">
      請輸入作者
    </small>
  </div>

  <div>
    <label>庫存數量</label>

    <input
      type="number"
      formControlName="stock">

    <small
      *ngIf="bookForm.get('stock').invalid &&
             bookForm.get('stock').touched">
      庫存數量不可小於 0
    </small>
  </div>

  <button
    type="submit"
    [disabled]="bookForm.invalid">
    儲存
  </button>
</form>
```

資料流：

```
queryBookList()
  → 選擇書本
  → query(bookId)
  → patchValue(book)
  → 修改 bookForm
  → save(bookForm.value)
```

---

# 7. 考試快速查找

| 想找的內容 | 位置 |
| --- | --- |
| `HttpClient.post()` | 第 2 節 |
| API Service | 第 2 節 |
| `HttpClientModule` | 第 3 節 |
| lazy loading | 第 4.3 節 |
| `RouterModule.forChild()` | 第 4.1 節 |
| Template-driven Form | 第 5 節 |
| `NgForm`、`ngModel` | 第 5 節 |
| Reactive Form | 第 6 節 |
| `FormGroup`、`patchValue()` | 第 6 節 |

## 常見錯誤

### `No provider for HttpClient`

檢查 `AppModule` 是否匯入 `HttpClientModule`。

### `Can't bind to 'ngModel'`

檢查功能 Module 是否匯入 `FormsModule`。

### `Can't bind to 'formGroup'`

檢查功能 Module 是否匯入 `ReactiveFormsModule`。

### API 有資料，但 Reactive Form 沒顯示

確認 API 成功後有執行：

```tsx
this.bookForm.patchValue(book);
```

### 儲存時沒有書本 ID

確認 `bookForm` 包含 `bookId`：

```tsx
bookId: [null]
```