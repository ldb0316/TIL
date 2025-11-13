Quick sort는 배열에서 현재 기준이 되는 값보다 작은 값을 왼쪽으로, 큰 값을 오른쪽으로 전부 밀어서 정리하고
기준이 되는 값이 그 중간으로 위치를 옮겨가는것을 재귀적으로 반복하며 정렬한다.
기준이 되는 값을 pivot value라 하며, pivot value가 현재 배열에서 존재하는 index가 pivot index이다.

시간 복잡도는 O(nlogn) ~ O(n^2) 이며, 이미 정렬이 완료되어있는 배열에서는 O(n^2)가 된다. 

``` java
public void quickSort(int[] arr, int startIndex, int endIndex) {
	if(startIndex >= endIndex) return; // 시작index 와 끝index가 일치하거나 start가 더 크면 종료(재귀탈출)
	
	int pivotIndex = partition(arr, startIndex, endIndex); //좌우 값 배치 후 pivotIndex return
	quickSort(arr, startIndex, pivotIndex - 1); // pivot index 기준 좌측 (작은값들의 배열) 다시 재귀호출
	quickSort(arr, pivotIndex + 1, endIndex); // pivot index 기준 우측 (큰값들의 배열) 다시 재귀호출
}

public int partition(int[] arr, int startIndex, int endIndex) {
	int pivotValue = arr[endIndex]; // endIndex에 해당하는 값 (last value)를 pivot Value로 잡는다.
	int i = startIndex - 1; //장래에 pivot index가 될 친구이다.
	
	for(int j=startIndex; j<endIndex; j++) { // 시작 index부터 끝 index의 직전까지(pivotValue는 제외하기위해)
		// pivotValue보다 큰 값이 있다면 i는 현재값을 유지하고 다음 반복문으로 넘어가기때문에 j는 커진다.
		// 즉, 일시적으로 i는 pivotValue보다 큰 값의 index가 된다. 
		if(arr[j] < pivotValue) { // pivotValue보다 작은 값이 존재한다면
			i++;
			swap(arr, i, j) // i와 j에 차이가 있다면 i (pivotValue보다 큰 값의 인덱스) 와 j (pivotValue보다 작은 값의 인덱스) 로 위치를 서로 교환한다.
		}
	}
	
	swap(arr, i+1, endIndex); // 마지막으로 교환 된 pivotValue보다 큰 값의 index의 다음 칸에 pivotValue 자신이 위치를 교환한다.
	// 이 시점에 배열에는 {작은값, 작은값, 작은값, pivotValue, 큰값, 큰값, 큰값} 과 같이 정렬되어있다.
	return i+1; // pivotValue의 현재 위치를 return 한다 (pivot index)
}

public void swap(int[] arr, int from, int to) {
	int temp = arr[to];
	arr[to] = arr[from];
	arr[from] = temp;
}

```