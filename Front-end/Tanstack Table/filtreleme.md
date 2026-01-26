Temel olarak filtreleme yapısı şu 3 ana bacağa oturur:

1. State Management: Filtre değerlerinin saklanması.
2. Filter Functions: Verinin nasıl eleneceğinin belirlenmesi.
3. UI Implementation: Kullanıcıdan girdinin alınıp tabloya iletilmesi.

# Temel Kurulum
Filtrelemeyi aktif etmek için useReactTable hook'una getFilteredRowModel fonksiyonunu ve state'i eklemelisiniz.

```tsx
import {
  useReactTable,
  getCoreRowModel,
  getFilteredRowModel, // 1. Bunu import et
  ColumnFiltersState,
} from '@tanstack/react-table'
import { useState } from 'react'

function MyTable({ data, columns }) {
  // 2. Filtre state'ini tutacak değişken. Tanstack'ten ColumnFiltersState tipini kullan.
  const [columnFilters, setColumnFilters] = useState<ColumnFiltersState>([])

  const table = useReactTable({
    data,
    columns,
    state: {
      columnFilters, // 3. State'i tabloya bağla
    },
    onColumnFiltersChange: setColumnFilters, // 4. State değişimini tabloya bildir
    getCoreRowModel: getCoreRowModel(),
    getFilteredRowModel: getFilteredRowModel(), // 5. Filtreleme motorunu aktif et
    getFacetedUniqueValues: getFacetedUniqueValues(), // 6. Faceted (Seçenekli) filtreleme için
  })

  // ... render işlemleri
}
```

# Sütun Bazlı Filtreleme
Her sütun için ayrı bir filtreleme yapmak isterseniz genellikle header bileşeninin içine