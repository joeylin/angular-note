#������֮���ͨ��

***

##����������ļ̳з�ʽ

����������ļ̳��ǻ���js��ԭ�ͼ̳з�ʽ�����������Ϊ����������������������ֵΪ�������͵�ʱ���޸ĸ������������ֵ��Ӱ�쵽�������򣬷�֮���޸���������ֻ��Ӱ�����������ֵ������Ӱ�츸�����������ֵ�������Ҫ��������������������һ��ֵ�Ļ�������Ҫ�õ�����һ�֣����������ϵ�ֵΪ�����κ�һ�����޸Ķ���Ӱ����һ����������Ϊ��js�ж������������͡�
    

�������ͣ�
```javascript
function Sandcrawler($scope) {
    $scope.location = "Mos Eisley North";
    $scope.move = function(newLocation) {
        $scope.location = newLocation;
    }
}
function Droid($scope) {
    $scope.sell = function(newLocation) {
        $scope.location = newLocation;
    }
}
// html
<div ng-controller="Sandcrawler">
    <p>Location: {{location}}</p>
    <button ng-click="move('Mos Eisley South')">Move</button>
    <div ng-controller="Droid">
        <p>Location: {{location}}</p>
        <button ng-click="sell('Owen Farm')">Sell</button>
    </div>
</div>

```

�������ͣ�
```javascript
function Sandcrawler($scope) {
    $scope.location = "Mos Eisley North";
    $scope.move = function(newLocation) {
        $scope.location = newLocation;
    }
}
function Droid($scope) {
    $scope.sell = function(newLocation) {
        $scope.location = newLocation;
    }
}
// html
<div ng-controller="Sandcrawler">
    <p>Sandcrawler Location: {{location}}</p>
    <div ng-controller="Droid">
        <button ng-click="summon('Owen Farm')">
            Summon Sandcrawler
        </button>
    </div>
</div>
```

##�����¼��ķ�ʽ

��һ������»��ڼ̳еķ�ʽ�Ѿ��㹻����󲿷�����ˣ��������ַ�ʽû��ʵ���ֵܿ�����֮���ͨ�ŷ�ʽ�������������¼��ķ�ʽ�������¼��ķ�ʽ�����ǿ����������õ�$on,$emit,$boardcast�⼸����ʽ��ʵ�֣�����$on��ʾ�¼�������$emit��ʾ�򸸼����ϵ������򴥷��¼��� $boardcast��ʾ���Ӽ����µ�������㲥�¼����������´��룺

���ϴ����¼���
```javascript
function Sandcrawler($scope) {
    $scope.sandcrawler.location = "Mos Eisley North";
}
function Droid($scope) {
    $scope.summon = function(newLocation) {
        $scope.sandcrawler.location = newLocation;
    }
}
// html
<div ng-controller="Sandcrawler">
    <p>Sandcrawler Location: {{location}}</p>
    <div ng-controller="Droid">
        <button ng-click="summon('Owen Farm')">
            Summon Sandcrawler
        </button>
    </div>
</div>
```

���¹㲥�¼�
```javascript
function Sandcrawler($scope) {
    $scope.location = "Mos Eisley North";
    $scope.recall = function() {
        $scope.$broadcast('recall', $scope.location);
    }
}
function Droid($scope) {
    $scope.location = "Owen Farm";
    $scope.$on('recall', function(e, newLocation) {
        $scope.location = newLocation;
    });
}
//html
<div ng-controller="Sandcrawler">
    <p>Sandcrawler Location: {{location}}</p>
    <button ng-click="recall()">Recall Droids</button>
    <div ng-controller="Droid">
        <p>Droid Location: {{location}}</p>
    </div>
</div>

```

������÷����ǿ��������һ�������ֵܿ��Ƽ����ͨ�ŵķ�������������һ���ֵܿ������������򴥷�һ���¼���Ȼ���ڸ��������м����¼����ٹ㲥��������������ͨ���¼�Я���Ĳ�����ʵ�������ݾ��������������ֵ�������֮�䴫�����뿴���룺

�ֵ�������֮�䴫����
```javascript
function Sandcrawler($scope) {
    $scope.$on('requestDroidRecall', function(e) {
        $scope.$broadcast('executeDroidRecall');
    });
}
function Droid($scope) {
    $scope.location = "Owen Farm";
    $scope.recallAllDroids = function() {
        $scope.$emit('requestDroidRecall');
    }
    $scope.$on('executeDroidRecall', function() { 
        $scope.location = "Sandcrawler"
    });
}
// html
<div ng-controller="Sandcrawler">
    <div ng-controller="Droid">
        <h2>R2-D2</h2>
        <p>Droid Location: {{location}}</p>
        <button ng-click="recallAddDroids()">Recall All Droids</button>
    </div>
    <div ng-controller="Droid">
        <h2>C-3PO</h2>
        <p>Droid Location: {{status}}</p>
        <button ng-click="recallAddDroids()">Recall All Droids</button>
    </div>
</div>
```


