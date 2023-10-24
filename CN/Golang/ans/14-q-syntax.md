---
title: '判断切片是否相同的两种方式性能比较'
---

比较两个切片可以使用两种方式：

1. 遍历切片比较每个元素，可以判断临界条件以快速失败提高性能
2. 使用反射，`reflect.DeepEqual(x, y any) bool`，因为是通用型函数，并且使用反射获取类型信息，在有性能要求的场景中不建议使用

## 1. 实现

```go
func sliceEqual(a1, a2 []int) bool {
	if len(a1) != len(a2) {
		return false
	}

	// 异或运算
	// 参与比较的切片不能为 nil
	if (a1 == nil) != (a2 == nil) {
		return false
	}

	for i, v := range a1 {
		if a2[i] != v {
			return false
		}
	}

	return true
}
```

1. 判断临界条件：长度是否相同或比较的切片中含有 nil ，以快速失败提升性能
2. 遍历比较每个元素

### 1.1 单元测试

```go
func Test_sliceEqual(t *testing.T) {
    type args struct {
       a1 []int
       a2 []int
    }
    tests := []struct {
       name string
       args args
       want bool
    }{
       // TODO: Add test cases.
       {name: "", args: args{a1: []int{}, a2: make([]int, 0)}, want: true},
       {name: "", args: args{a1: []int(nil), a2: make([]int, 0)}, want: false},
       {name: "", args: args{a1: make([]int, 0), a2: []int(nil)}, want: false},
       {name: "", args: args{a1: []int(nil), a2: []int(nil)}, want: true},
       {name: "", args: args{a1: []int{}, a2: []int{}}, want: true},
       {name: "", args: args{a1: []int{1, 2, 3}, a2: []int{1, 2, 3}}, want: true},
       {name: "", args: args{a1: []int{1}, a2: []int{1, 2, 3}}, want: false},
       {name: "", args: args{a1: []int{}, a2: []int{1, 2, 3}}, want: false},
       {name: "", args: args{a1: []int(nil), a2: []int{1, 2, 3}}, want: false},
    }
    for _, tt := range tests {
       t.Run(tt.name, func(t *testing.T) {
          if got := sliceEqual(tt.args.a1, tt.args.a2); got != tt.want {
             t.Errorf("sliceEqual() = %v, want %v", got, tt.want)
          }
       })
    }
}
```

## 2. Benchmark

测试用例使用时间复杂度最差的情景，即两个非空且元素完全相同的切片。

```go
func genSlice(val, n int) []int {
	a := make([]int, n)
	for i := 0; i < n; i++ {
		a[i] = val
	}

	return a
}

func BenchmarkSliceEqual(b *testing.B) {
	tests := []struct {
		name string
		f    func(a1, a2 []int) bool
	}{
		{name: "UsingIteration", f: sliceEqual},
		{name: "UsingReflect", f: func(a1, a2 []int) bool {
			return reflect.DeepEqual(a1, a2)
		}},
	}

	for k := 10; k <= 100000; k *= 10 {
		val := rand.Int()

		a1 := genSlice(val, k)
		a2 := genSlice(val, k)

		for _, tt := range tests {
			b.Run(fmt.Sprintf("%-15s_%.0e", tt.name, float64(k)), func(b *testing.B) {
				for i := 0; i < b.N; i++ {
					tt.f(a1, a2)
				}
			})
		}
	}
}
```

```
BenchmarkSliceEqual/UsingIteration__1e+01-12            212428482                5.885 ns/op           0 B/op          0 allocs/op
BenchmarkSliceEqual/UsingReflect____1e+01-12             3348832               301.5 ns/op            48 B/op          2 allocs/op
BenchmarkSliceEqual/UsingIteration__1e+02-12            25925092                45.72 ns/op            0 B/op          0 allocs/op
BenchmarkSliceEqual/UsingReflect____1e+02-12              752869              1548 ns/op              48 B/op          2 allocs/op
BenchmarkSliceEqual/UsingIteration__1e+03-12             3597279               328.5 ns/op             0 B/op          0 allocs/op
BenchmarkSliceEqual/UsingReflect____1e+03-12               86875             13862 ns/op              48 B/op          2 allocs/op
BenchmarkSliceEqual/UsingIteration__1e+04-12              295510              3861 ns/op               0 B/op          0 allocs/op
BenchmarkSliceEqual/UsingReflect____1e+04-12                7898            138434 ns/op              48 B/op          2 allocs/op
BenchmarkSliceEqual/UsingIteration__1e+05-12               28862             44241 ns/op               0 B/op          0 allocs/op
BenchmarkSliceEqual/UsingReflect____1e+05-12                 879           1367174 ns/op              48 B/op          2 allocs/op
```

可以看出，直接使用遍历的方式时间性能约为使用反射的约30倍。

## Reference

1. https://stackoverflow.com/questions/23025694/is-there-no-xor-operator-for-booleans-in-golang
2. https://stackoverflow.com/questions/15311969/checking-the-equality-of-two-slices

