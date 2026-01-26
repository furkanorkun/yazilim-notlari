# TanStack Table ile Client-Side Query Parameters Yönetimi

Bu makalede, TanStack Table ile oluşturduğunuz tablolarda filtreleme, sıralama, sayfalama gibi tüm state'leri URL query parametrelerinde tutmayı ve tarayıcıda ileri-geri butonlarının da çalışmasını sağlamayı öğreneceğiz. Vite.js ve React Router kullanarak adım adım implementasyonu yapacağız.

## 🎯 Hedeflerimiz

- ✅ Tablo state'lerini URL'de saklamak (filtreleme, sıralama, sayfalama)
- ✅ Sayfa yenilense de filtrelerin korunması
- ✅ Tarayıcı ileri-geri butonlarının çalışması
- ✅ URL'yi paylaşabilme (bookmark yapabilme)
- ✅ Client-side routing ile smooth geçişler
- ✅ Type-safe URL parameter yönetimi

## 📋 Gereksinimler

```bash
# Temel paketler
npm create vite@latest my-table-app -- --template react-ts
cd my-table-app
npm install

# TanStack Table ve UI bileşenleri
npm install @tanstack/react-table
npm install react-router-dom
npm install lucide-react

# Shadcn UI bileşenleri (opsiyonel ama önerilen)
npx shadcn-ui@latest init
npx shadcn-ui@latest add table button input select checkbox dropdown-menu command popover separator badge
```

## 🏗️ Proje Yapısı

```
src/
├── components/
│   └── ui/                          # Shadcn bileşenleri
│   └── data-table/                  # Tablo bileşenleri
│       ├── data-table.tsx
│       ├── data-table-pagination.tsx
│       ├── data-table-toolbar.tsx
│       └── data-table-faceted-filter.tsx
├── hooks/
│   └── use-table-url-state.ts      # 🔥 URL state management hook'u
├── lib/
│   ├── utils.ts
│   └── table-url-utils.ts           # 🔥 URL utility fonksiyonları
├── pages/
│   └── users/
│       ├── page.tsx
│       └── columns.tsx
└── App.tsx
```

## 🔧 1. URL Utility Fonksiyonları

Öncelikle URL parametrelerini yönetmek için utility fonksiyonlarımızı oluşturalım.

**Dosya:** `src/lib/table-url-utils.ts`

```typescript
import { ColumnFiltersState, SortingState, PaginationState } from "@tanstack/react-table"

export interface TableUrlState {
  page: number
  pageSize: number
  sorting: SortingState
  filters: ColumnFiltersState
  globalFilter: string
}

// URL'den parametreleri parse etme
export function parseTableStateFromUrl(searchParams: URLSearchParams): Partial<TableUrlState> {
  const state: Partial<TableUrlState> = {}

  // Sayfalama
  const page = searchParams.get('page')
  const pageSize = searchParams.get('pageSize')
  
  if (page) {
    state.page = Math.max(0, parseInt(page, 10) - 1) // URL'de 1-based, kod içinde 0-based
  }
  
  if (pageSize) {
    state.pageSize = parseInt(pageSize, 10)
  }

  // Sıralama
  const sortBy = searchParams.get('sortBy')
  const sortOrder = searchParams.get('sortOrder')
  
  if (sortBy && sortOrder) {
    state.sorting = [{
      id: sortBy,
      desc: sortOrder === 'desc'
    }]
  } else {
    state.sorting = []
  }

  // Global filtre (arama)
  const search = searchParams.get('search')
  if (search) {
    state.globalFilter = search
  }

  // Column filtreleri
  const filters: ColumnFiltersState = []
  
  // Status filtresi (çoklu seçim)
  const status = searchParams.get('status')
  if (status) {
    const statusValues = status.split(',').filter(Boolean)
    if (statusValues.length > 0) {
      filters.push({
        id: 'status',
        value: statusValues
      })
    }
  }

  // Diğer tekil filtreler
  const name = searchParams.get('name')
  if (name) {
    filters.push({
      id: 'name',
      value: name
    })
  }

  const email = searchParams.get('email')
  if (email) {
    filters.push({
      id: 'email',
      value: email
    })
  }

  state.filters = filters

  return state
}

// State'i URL parametrelerine dönüştürme
export function buildUrlFromTableState(
  baseUrl: string,
  state: Partial<TableUrlState>
): string {
  const url = new URL(baseUrl, window.location.origin)
  const params = url.searchParams

  // Önceki parametreleri temizle
  params.delete('page')
  params.delete('pageSize')
  params.delete('sortBy')
  params.delete('sortOrder')
  params.delete('search')
  params.delete('status')
  params.delete('name')
  params.delete('email')

  // Sayfalama
  if (state.page !== undefined && state.page > 0) {
    params.set('page', (state.page + 1).toString()) // 0-based'den 1-based'e çevir
  }

  if (state.pageSize !== undefined && state.pageSize !== 10) { // 10 varsayılan ise
    params.set('pageSize', state.pageSize.toString())
  }

  // Sıralama
  if (state.sorting && state.sorting.length > 0) {
    const sort = state.sorting[0]
    params.set('sortBy', sort.id)
    params.set('sortOrder', sort.desc ? 'desc' : 'asc')
  }

  // Global filtre
  if (state.globalFilter && state.globalFilter.trim()) {
    params.set('search', state.globalFilter.trim())
  }

  // Column filtreleri
  if (state.filters) {
    state.filters.forEach((filter) => {
      const { id, value } = filter

      if (Array.isArray(value) && value.length > 0) {
        // Çoklu seçim filtreleri (örn: status)
        params.set(id, value.join(','))
      } else if (typeof value === 'string' && value.trim()) {
        // Tekil filtreler
        params.set(id, value.trim())
      }
    })
  }

  return url.pathname + url.search
}

// Default table state'i
export const defaultTableState: TableUrlState = {
  page: 0,
  pageSize: 10,
  sorting: [],
  filters: [],
  globalFilter: ''
}
```

## 🪝 2. Custom URL State Hook'u

Şimdi tablo state'ini URL ile senkronize eden React hook'umuzu oluşturalım.

**Dosya:** `src/hooks/use-table-url-state.ts`

```typescript
import { useCallback, useMemo } from 'react'
import { useSearchParams, useNavigate } from 'react-router-dom'
import { 
  ColumnFiltersState, 
  SortingState, 
  PaginationState 
} from '@tanstack/react-table'
import { 
  parseTableStateFromUrl, 
  buildUrlFromTableState, 
  defaultTableState,
  type TableUrlState 
} from '@/lib/table-url-utils'

export function useTableUrlState() {
  const [searchParams] = useSearchParams()
  const navigate = useNavigate()

  // URL'den mevcut state'i parse et
  const currentState = useMemo(() => {
    const urlState = parseTableStateFromUrl(searchParams)
    return {
      ...defaultTableState,
      ...urlState
    }
  }, [searchParams])

  // URL'yi güncelleme fonksiyonu
  const updateUrl = useCallback((newState: Partial<TableUrlState>) => {
    const mergedState = {
      ...currentState,
      ...newState
    }

    const newUrl = buildUrlFromTableState(window.location.pathname, mergedState)
    
    // Replace yerine push kullanarak history'de gezinebilirlik sağla
    navigate(newUrl, { replace: false })
  }, [currentState, navigate])

  // Pagination state'ini güncelleme
  const updatePagination = useCallback((pagination: PaginationState) => {
    updateUrl({
      page: pagination.pageIndex,
      pageSize: pagination.pageSize
    })
  }, [updateUrl])

  // Sorting state'ini güncelleme
  const updateSorting = useCallback((sorting: SortingState) => {
    updateUrl({
      sorting,
      page: 0 // Sıralama değiştiğinde ilk sayfaya git
    })
  }, [updateUrl])

  // Column filters'ı güncelleme
  const updateColumnFilters = useCallback((filters: ColumnFiltersState) => {
    updateUrl({
      filters,
      page: 0 // Filtre değiştiğinde ilk sayfaya git
    })
  }, [updateUrl])

  // Global filter'ı güncelleme
  const updateGlobalFilter = useCallback((globalFilter: string) => {
    updateUrl({
      globalFilter,
      page: 0 // Arama değiştiğinde ilk sayfaya git
    })
  }, [updateUrl])

  // Tüm filtreleri temizleme
  const clearAllFilters = useCallback(() => {
    updateUrl({
      filters: [],
      globalFilter: '',
      page: 0
    })
  }, [updateUrl])

  return {
    // Current state
    pagination: {
      pageIndex: currentState.page,
      pageSize: currentState.pageSize
    },
    sorting: currentState.sorting,
    columnFilters: currentState.filters,
    globalFilter: currentState.globalFilter,

    // Update functions
    updatePagination,
    updateSorting,
    updateColumnFilters,
    updateGlobalFilter,
    clearAllFilters
  }
}
```

## 🔄 3. Data Table Bileşenini Güncelleme

Artık ana tablo bileşenimizi URL state'i kullanacak şekilde güncelleyelim.

**Dosya:** `src/components/data-table/data-table.tsx`

```typescript
import * as React from "react"
import {
  ColumnDef,
  ColumnFiltersState,
  SortingState,
  VisibilityState,
  flexRender,
  getCoreRowModel,
  getFacetedRowModel,
  getFacetedUniqueValues,
  getFilteredRowModel,
  getPaginationRowModel,
  getSortedRowModel,
  useReactTable,
} from "@tanstack/react-table"

import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table"

import { DataTablePagination } from "./data-table-pagination"
import { DataTableToolbar } from "./data-table-toolbar"
import { useTableUrlState } from "@/hooks/use-table-url-state"

interface DataTableProps<TData, TValue> {
  columns: ColumnDef<TData, TValue>[]
  data: TData[]
}

export function DataTable<TData, TValue>({
  columns,
  data,
}: DataTableProps<TData, TValue>) {
  // 🔥 URL state hook'unu kullan
  const {
    pagination,
    sorting,
    columnFilters,
    globalFilter,
    updatePagination,
    updateSorting,
    updateColumnFilters,
    updateGlobalFilter,
  } = useTableUrlState()

  // Local state'ler (URL'de tutmadığımız)
  const [rowSelection, setRowSelection] = React.useState({})
  const [columnVisibility, setColumnVisibility] = React.useState<VisibilityState>({})

  const table = useReactTable({
    data,
    columns,
    state: {
      sorting,
      columnVisibility,
      rowSelection,
      columnFilters,
      pagination,
      globalFilter,
    },
    enableRowSelection: true,
    onRowSelectionChange: setRowSelection,
    onSortingChange: updateSorting,           // 🔥 URL'e bağla
    onColumnFiltersChange: updateColumnFilters, // 🔥 URL'e bağla
    onColumnVisibilityChange: setColumnVisibility,
    onPaginationChange: updatePagination,     // 🔥 URL'e bağla
    onGlobalFilterChange: updateGlobalFilter, // 🔥 URL'e bağla
    getCoreRowModel: getCoreRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFacetedRowModel: getFacetedRowModel(),
    getFacetedUniqueValues: getFacetedUniqueValues(),
    manualPagination: false, // Client-side pagination
    manualSorting: false,    // Client-side sorting
    manualFiltering: false,  // Client-side filtering
  })

  return (
    <div className="space-y-4">
      {/* TOOLBAR */}
      <DataTableToolbar table={table} />

      {/* TABLE */}
      <div className="rounded-md border">
        <Table>
          <TableHeader>
            {table.getHeaderGroups().map((headerGroup) => (
              <TableRow key={headerGroup.id}>
                {headerGroup.headers.map((header) => {
                  return (
                    <TableHead key={header.id} colSpan={header.colSpan}>
                      {header.isPlaceholder
                        ? null
                        : flexRender(
                            header.column.columnDef.header,
                            header.getContext()
                          )}
                    </TableHead>
                  )
                })}
              </TableRow>
            ))}
          </TableHeader>
          <TableBody>
            {table.getRowModel().rows?.length ? (
              table.getRowModel().rows.map((row) => (
                <TableRow
                  key={row.id}
                  data-state={row.getIsSelected() && "selected"}
                >
                  {row.getVisibleCells().map((cell) => (
                    <TableCell key={cell.id}>
                      {flexRender(
                        cell.column.columnDef.cell,
                        cell.getContext()
                      )}
                    </TableCell>
                  ))}
                </TableRow>
              ))
            ) : (
              <TableRow>
                <TableCell
                  colSpan={columns.length}
                  className="h-24 text-center"
                >
                  Sonuç bulunamadı.
                </TableCell>
              </TableRow>
            )}
          </TableBody>
        </Table>
      </div>

      {/* PAGINATION */}
      <DataTablePagination table={table} />
    </div>
  )
}
```

## 🛠️ 4. Toolbar Bileşenini Güncelleme

Toolbar'da global arama ve filtre temizleme işlevlerini de URL ile senkronize edelim.

**Dosya:** `src/components/data-table/data-table-toolbar.tsx`

```typescript
import { Table } from "@tanstack/react-table"
import { X } from "lucide-react"

import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { DataTableFacetedFilter } from "./data-table-faceted-filter"
import { useTableUrlState } from "@/hooks/use-table-url-state"

// Filtre seçenekleri
export const statuses = [
  { value: "active", label: "Active" },
  { value: "inactive", label: "Inactive" },
  { value: "pending", label: "Pending" },
]

interface DataTableToolbarProps<TData> {
  table: Table<TData>
}

export function DataTableToolbar<TData>({
  table,
}: DataTableToolbarProps<TData>) {
  const { clearAllFilters } = useTableUrlState()
  
  const isFiltered = table.getState().columnFilters.length > 0 || 
                    !!table.getState().globalFilter

  return (
    <div className="flex items-center justify-between">
      <div className="flex flex-1 items-center space-x-2">
        {/* GLOBAL SEARCH - URL'deki 'search' parametresiyle senkronize */}
        <Input
          placeholder="Tüm alanlarda ara..."
          value={table.getState().globalFilter ?? ""}
          onChange={(event) => table.setGlobalFilter(event.target.value)}
          className="h-8 w-[150px] lg:w-[250px]"
        />

        {/* NAME FILTER - Spesifik column filtresi */}
        <Input
          placeholder="İsim filtrele..."
          value={(table.getColumn("name")?.getFilterValue() as string) ?? ""}
          onChange={(event) =>
            table.getColumn("name")?.setFilterValue(event.target.value)
          }
          className="h-8 w-[150px] lg:w-[200px]"
        />

        {/* STATUS FACETED FILTER */}
        {table.getColumn("status") && (
          <DataTableFacetedFilter
            column={table.getColumn("status")}
            title="Durum"
            options={statuses}
          />
        )}

        {/* CLEAR ALL FILTERS */}
        {isFiltered && (
          <Button
            variant="ghost"
            onClick={clearAllFilters} // 🔥 URL'deki tüm filtreleri temizle
            className="h-8 px-2 lg:px-3"
          >
            Filtreleri Sıfırla
            <X className="ml-2 h-4 w-4" />
          </Button>
        )}
      </div>

      {/* VIEW OPTIONS (opsiyonel) */}
      <div className="flex items-center space-x-2">
        <p className="text-sm text-muted-foreground">
          {table.getFilteredRowModel().rows.length} sonuç
        </p>
      </div>
    </div>
  )
}
```

## 🔀 5. React Router Setup'ı

Ana App.tsx dosyamızı React Router ile yapılandıralım.

**Dosya:** `src/App.tsx`

```typescript
import { BrowserRouter as Router, Routes, Route, Navigate } from 'react-router-dom'
import UsersPage from './pages/users/page'
import './App.css'

function App() {
  return (
    <Router>
      <div className="min-h-screen bg-background">
        <header className="border-b">
          <div className="container mx-auto px-4 py-4">
            <h1 className="text-2xl font-bold">TanStack Table ile URL State Management</h1>
          </div>
        </header>
        
        <main className="container mx-auto px-4 py-8">
          <Routes>
            <Route path="/" element={<Navigate to="/users" replace />} />
            <Route path="/users" element={<UsersPage />} />
          </Routes>
        </main>
      </div>
    </Router>
  )
}

export default App
```

## 👥 6. Users Sayfası (Final Implementation)

Şimdi tüm parçaları bir araya getirip users sayfasını oluşturalım.

**Dosya:** `src/pages/users/page.tsx`

```typescript
import { User, columns } from "./columns"
import { DataTable } from "@/components/data-table/data-table"

// Mock data - gerçek projede API'den gelecek
async function getData(): Promise<User[]> {
  return [
    {
      id: "728ed52f",
      amount: 100,
      status: "pending",
      email: "ahmet@example.com",
      name: "Ahmet Yılmaz"
    },
    {
      id: "489e1d42",
      amount: 250,
      status: "active", 
      email: "zeynep@example.com",
      name: "Zeynep Kaya"
    },
    {
      id: "619e1d52",
      amount: 175,
      status: "inactive",
      email: "mehmet@example.com", 
      name: "Mehmet Demir"
    },
    {
      id: "719f2e63",
      amount: 300,
      status: "active",
      email: "ayse@example.com",
      name: "Ayşe Özkan"
    },
    {
      id: "829g3f74",
      amount: 450,
      status: "pending",
      email: "fatma@example.com",
      name: "Fatma Şen"
    },
    {
      id: "939h4g85",
      amount: 125,
      status: "active",
      email: "ali@example.com",
      name: "Ali Çelik"
    },
    {
      id: "049i5h96",
      amount: 275,
      status: "inactive",
      email: "elif@example.com",
      name: "Elif Aydın"
    },
    {
      id: "159j6i07",
      amount: 350,
      status: "active",
      email: "murat@example.com",
      name: "Murat Koç"
    },
    {
      id: "269k7j18",
      amount: 200,
      status: "pending",
      email: "seda@example.com",
      name: "Seda Yıldız"
    },
    {
      id: "379l8k29",
      amount: 425,
      status: "active",
      email: "burak@example.com",
      name: "Burak Arslan"
    },
    {
      id: "489m9l30",
      amount: 150,
      status: "inactive",
      email: "deniz@example.com",
      name: "Deniz Güler"
    },
    {
      id: "599n0m41",
      amount: 375,
      status: "active",
      email: "cemre@example.com",
      name: "Cemre Öztürk"
    }
  ]
}

export default function UsersPage() {
  // Client-side data - API call'u burada yapabilirsiniz
  const [data, setData] = useState<User[]>([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    getData().then((fetchedData) => {
      setData(fetchedData)
      setLoading(false)
    })
  }, [])

  if (loading) {
    return (
      <div className="flex items-center justify-center h-64">
        <div className="text-lg">Yükleniyor...</div>
      </div>
    )
  }

  return (
    <div className="space-y-6">
      <div>
        <h1 className="text-3xl font-bold tracking-tight">Kullanıcı Listesi</h1>
        <p className="text-muted-foreground">
          TanStack Table ile URL state management örneği
        </p>
      </div>
      
      <DataTable columns={columns} data={data} />
    </div>
  )
}
```

**Dosya:** `src/pages/users/columns.tsx`

```typescript
import { ColumnDef } from "@tanstack/react-table"
import { ArrowUpDown, MoreHorizontal } from "lucide-react"
import { Button } from "@/components/ui/button"
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuLabel,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu"
import { Checkbox } from "@/components/ui/checkbox"
import { Badge } from "@/components/ui/badge"

export type User = {
  id: string
  name: string
  email: string
  status: "active" | "inactive" | "pending"
  amount: number
}

export const columns: ColumnDef<User>[] = [
  // SELECTION COLUMN
  {
    id: "select",
    header: ({ table }) => (
      <Checkbox
        checked={table.getIsAllPageRowsSelected()}
        onCheckedChange={(value) => table.toggleAllPageRowsSelected(!!value)}
        aria-label="Select all"
      />
    ),
    cell: ({ row }) => (
      <Checkbox
        checked={row.getIsSelected()}
        onCheckedChange={(value) => row.toggleSelected(!!value)}
        aria-label="Select row"
      />
    ),
    enableSorting: false,
    enableHiding: false,
  },

  // NAME COLUMN - Sortable ve filterable
  {
    accessorKey: "name",
    header: ({ column }) => {
      return (
        <Button
          variant="ghost"
          onClick={() => column.toggleSorting(column.getIsSorted() === "asc")}
        >
          İsim
          <ArrowUpDown className="ml-2 h-4 w-4" />
        </Button>
      )
    },
  },

  // EMAIL COLUMN - Sortable
  {
    accessorKey: "email", 
    header: ({ column }) => {
      return (
        <Button
          variant="ghost"
          onClick={() => column.toggleSorting(column.getIsSorted() === "asc")}
        >
          Email
          <ArrowUpDown className="ml-2 h-4 w-4" />
        </Button>
      )
    },
  },

  // STATUS COLUMN - Faceted filter için önemli
  {
    accessorKey: "status",
    header: "Durum",
    cell: ({ row }) => {
      const status = row.getValue("status") as string
      
      const statusConfig = {
        active: { label: "Aktif", variant: "default" as const },
        inactive: { label: "Pasif", variant: "secondary" as const },
        pending: { label: "Bekliyor", variant: "outline" as const },
      }
      
      const config = statusConfig[status as keyof typeof statusConfig]
      
      return (
        <Badge variant={config.variant}>
          {config.label}
        </Badge>
      )
    },
    filterFn: (row, id, value) => {
      return value.includes(row.getValue(id))
    },
  },

  // AMOUNT COLUMN - Sortable ve formatted
  {
    accessorKey: "amount",
    header: ({ column }) => {
      return (
        <div className="text-right">
          <Button
            variant="ghost"
            onClick={() => column.toggleSorting(column.getIsSorted() === "asc")}
          >
            Tutar
            <ArrowUpDown className="ml-2 h-4 w-4" />
          </Button>
        </div>
      )
    },
    cell: ({ row }) => {
      const amount = parseFloat(row.getValue("amount"))
      const formatted = new Intl.NumberFormat("tr-TR", {
        style: "currency",
        currency: "TRY",
      }).format(amount)

      return <div className="text-right font-medium">{formatted}</div>
    },
  },

  // ACTIONS COLUMN
  {
    id: "actions",
    cell: ({ row }) => {
      const user = row.original

      return (
        <DropdownMenu>
          <DropdownMenuTrigger asChild>
            <Button variant="ghost" className="h-8 w-8 p-0">
              <span className="sr-only">Menüyü aç</span>
              <MoreHorizontal className="h-4 w-4" />
            </Button>
          </DropdownMenuTrigger>
          <DropdownMenuContent align="end">
            <DropdownMenuLabel>İşlemler</DropdownMenuLabel>
            <DropdownMenuItem onClick={() => navigator.clipboard.writeText(user.id)}>
              ID Kopyala
            </DropdownMenuItem>
            <DropdownMenuItem>Kullanıcıyı Düzenle</DropdownMenuItem>
          </DropdownMenuContent>
        </DropdownMenu>
      )
    },
  },
]
```

## ⚡ 7. Performans Optimizasyonları

### Debounced Search

Global arama için debounce ekleyerek performansı artıralım.

**Dosya:** `src/hooks/use-debounced-value.ts`

```typescript
import { useEffect, useState } from 'react'

export function useDebouncedValue<T>(value: T, delay: number = 500) {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(timer)
  }, [value, delay])

  return debouncedValue
}
```

Toolbar'da kullanım:

```typescript
// data-table-toolbar.tsx içinde
import { useDebouncedValue } from "@/hooks/use-debounced-value"

export function DataTableToolbar<TData>({ table }: DataTableToolbarProps<TData>) {
  const [searchValue, setSearchValue] = useState(table.getState().globalFilter ?? "")
  const debouncedSearchValue = useDebouncedValue(searchValue, 300)
  
  // Debounced value URL'e yansısın
  useEffect(() => {
    table.setGlobalFilter(debouncedSearchValue)
  }, [debouncedSearchValue, table])

  return (
    <div className="flex items-center justify-between">
      <div className="flex flex-1 items-center space-x-2">
        <Input
          placeholder="Tüm alanlarda ara..."
          value={searchValue}
          onChange={(event) => setSearchValue(event.target.value)}
          className="h-8 w-[150px] lg:w-[250px]"
        />
        {/* Diğer filtreler... */}
      </div>
    </div>
  )
}
```

## 🧪 8. Test Senaryoları

URL state management'ın düzgün çalıştığını test etmek için:

### Manuel Test Listesi

1. **Sayfalama Testi**
   - ✅ Sayfa değiştir → URL'de `page` parametresi güncellenmeli
   - ✅ Sayfa boyutunu değiştir → URL'de `pageSize` parametresi güncellenmeli
   - ✅ Sayfayı yenile → aynı sayfa ve boyutta kalmalı

2. **Sıralama Testi**
   - ✅ Sütun başlığına tıkla → URL'de `sortBy` ve `sortOrder` parametreleri eklenmeli
   - ✅ Tersine sırala → `sortOrder` `asc/desc` arasında geçmeli
   - ✅ Sayfayı yenile → sıralama korunmalı

3. **Filtreleme Testi**
   - ✅ Global arama yap → URL'de `search` parametresi eklenmeli
   - ✅ Sütun filtresi ekle → URL'de ilgili parametre eklenmeli
   - ✅ Faceted filter kullan → çoklu seçim URL'de virgülle ayrılmalı
   - ✅ Filtreleri temizle → URL'den parametreler kalkmalı

4. **Tarayıcı Testi**
   - ✅ İleri/Geri butonları → tablo state'i değişmeli
   - ✅ URL'yi kopyalayıp yeni sekmede aç → aynı durum korunmalı
   - ✅ URL'yi bookmark yap → bookmark'tan açtığında state korunmalı

## 🎉 Sonuç

Artık client-side TanStack Table'ınız URL query parametreleri ile tamamen senkronize! Bu implementasyon size şunları sağlıyor:

✅ **Shareable URLs**: Kullanıcılar filtrelenmiş tabloyu URL ile paylaşabilir
✅ **Bookmarkable**: Favori filtre kombinasyonları bookmark'lanabilir  
✅ **Browser Navigation**: İleri/geri butonları mükemmel çalışıyor
✅ **Refresh Persistence**: Sayfa yenilense de state korunuyor
✅ **Type Safety**: TypeScript ile tip güvenli URL parameter yönetimi
✅ **Performance**: Debounced search ve optimized re-renders
✅ **Modularity**: Hook'lar ve utility'ler yeniden kullanılabilir

### 💡 İleri Düzey Özellikler

Bu temel yapı üzerine şunları da ekleyebilirsiniz:

- **Saved Views**: Kullanıcıların filtre kombinasyonlarını kaydetmesi
- **Advanced Filters**: Date range, numeric range filtreleri
- **Column Presets**: Farklı görünüm modları
- **Export Functionality**: Filtrelenmiş veriyi CSV/Excel'e aktarma
- **Real-time Updates**: WebSocket ile canlı veri güncellemeleri

Bu yaklaşım ile profesyonel seviyede, kullanıcı deneyimi odaklı tablolar oluşturabilirsiniz! 🚀