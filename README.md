# ID3_tree
1. The ID3 tree is built based on maximum information gain. 

2. Some interfaces are provided for top-down pruning and traversing. 

3. Visualization can be realized through graphviz package and dot file printed.	
	

# interfaces are already included in TreeBuilder
class TreeBuilder():
    
    def entropy(self, dataset):  #calculate entropy of a group                                                  
        prob = dataset.rule104.mean()
        if (prob > 0) & (prob <1):
            Entropy = -prob*log(prob,2)-(1-prob)*log((1-prob),2)
            
        else:
            Entropy = 0
        return Entropy
    
    
 
    def feature_entropy(self,dataset,feature): #calculate weighted sum of entropy of each kind of value for a feature
    
        u = dataset[feature].unique()
        sub_en = 0
        for i in u:
            sub_grp = dataset[dataset[feature] == i]

            p = sub_grp.rule104.mean()

            if (p >0) & (p<1):
                grp_en = (-p*log(p,2)-(1-p)*log((1-p),2))*sub_grp.shape[0]/dataset.shape[0]
                sub_en = sub_en + grp_en

            else: 
                sub_en = sub_en + 0

        return sub_en
                
        
        
 
    def FindSplit(self, feature, dataset): #find the best point to split
        min_entropy = 1000
        for i in dataset[feature].unique():
            big = dataset[dataset[feature] <= i]
            small = dataset[dataset[feature] > i]
            
            Entropy = self.entropy(big)*big.shape[0]/dataset.shape[0]+self.entropy(small)*small.shape[0]/dataset.shape[0]
            if Entropy < min_entropy:
                best_point = i
                min_entropy = Entropy
        return best_point
        
        
    def find_best_feature(self, dataset):  #find the feature providing maximum reduce of entropy
        min_entropy = 1000000
        for i in dataset.columns.values:
            if i != 'rule104':
                if self.feature_entropy(dataset,i) < min_entropy:
                    best_fea = i
                    min_entropy = self.feature_entropy(dataset,i)
        print(best_fea)
        return best_fea
    
    def piggyTree(self, dataset): #fully grow the tree
        
        node = TreeNode(dataset) 
        node.bad_rate = dataset.rule104.mean()
        print(dataset.rule104.mean())
        
        node.group_entropy = self.entropy(dataset)
        node.group_size = dataset.shape[0]
        print(dataset.shape[0])
        
        if (dataset.shape[1] <= 1) | (dataset.shape[0] <= 0):
            return node
        
        feat = self.find_best_feature(dataset)
        point = self.FindSplit(feat, dataset)
        node.feature = feat
        node.threshold = point
        #node.rule = feat+' &le; '+str(point)
        
        left = df(dataset[dataset[feat] <= point])
        left.drop(feat,axis=1,inplace=True)
        right = df(dataset[dataset[feat] > point])
        right.drop(feat,axis=1,inplace=True)

        node.left = self.piggyTree(left)
        node.right = self.piggyTree(right)
        
        return node
            
    def piggyTree_apply(self,tree,data_apply,size_,thre_):   #traverse grown three and apply split regulation to another set of data
        
        node = tree
        node_apply = TreeNode(data_apply)
        node_apply.bad_rate = data_apply.rule104.mean()
        print(data_apply.rule104.mean())
        
        node_apply.group_entropy = self.entropy(data_apply)
        node_apply.group_size = data_apply.shape[0]
        print(data_apply.shape[0])

        if (data_apply.shape[1] <= 1) | (data_apply.shape[0] <= 0) :
            return node_apply
        
        if node.group_size == size_:
            feat = 'rule99'
            point = thre_

        else:
            
            feat = node.feature
            point = node.threshold
        
        
        node_apply.feature = feat
        node_apply.threshold = point
        
        left = node.left
        right = node.right

        left_apply = df(data_apply[data_apply[feat] <= point])
        left_apply.drop(feat,axis=1,inplace=True)
        right_apply = df(data_apply[data_apply[feat] > point])
        right_apply.drop(feat,axis=1,inplace=True)
        
        
        node_apply.left = self.piggyTree_apply(left,left_apply,size_,thre_)
        node_apply.right = self.piggyTree_apply(right,right_apply,size_,thre_)
        
        return node_apply

    def DFS_addnode(self,TREE): #deep first searching to add node id
         stack = [TREE]
         Id = 1
         while stack !=[]:
             cur = stack.pop()
             cur.node_id = Id
             Id = Id+1
             print(cur.node_id)
             
             if cur.left != None:
                 stack.append(cur.left)
                 cur.left.parent = cur.node_id
     
                 
             if cur.right != None:
                 stack.append(cur.right)
                 cur.right.parent = cur.node_id

    def DFS_search_leaf(self,TREE): #DFS to print statistics of each leaf
        stack = [TREE]
        while stack !=[]:
            cur = stack.pop()
            isleaf = (cur.left == None) & (cur.right == None)
            
            if isleaf:
                print('this is a leaf,node_id is: ',cur.node_id,'parent is: ',cur.parent)
                print('br is: ',cur.bad_rate)
                print('size is: ',cur.group_size)
                print('______________________________________________________________________')
            else:

                if cur.left != None:
                    stack.append(cur.left) 
                if cur.right != None:
                    stack.append(cur.right) 
                  
    def DFS_search_total_numBad(self,TREE,nodeid): #统计每个node坏样本个数

        stack = [TREE]
        result = {}
        size = {}

        while stack !=[]:
            cur = stack.pop()
            result[cur.node_id] = cur.group_size*cur.bad_rate
            size[cur.node_id] = cur.group_size


            if cur.left != None:
                stack.append(cur.left)

            if cur.right != None:
                stack.append(cur.right)
        total_numBad = 0
        totalsize = 0
        for i in nodeid:
            total_numBad += result[i]
            totalsize += size[i]
        return total_numBad,totalsize  

                 
    def DFS_manual_pruning(self,TREE,id): #mannual pruning on parent node
        stack = [TREE]

        while stack !=[]:
            cur = stack.pop()

            if (cur.node_id == id):
                cur.left = None
                cur.right = None
            else:

                if cur.left != None:
                    stack.append(cur.left)

                if cur.right != None:
                    stack.append(cur.right)
                     
    def DFS_calculate_entropy(self,TREE,dataset): #calculate weighted sum of entropy of all leaves
     
        stack = [TREE]
        leaf_etp = 0

        while stack !=[]:
            cur = stack.pop()
            isleaf = (cur.left == None) & (cur.right == None)  
            if isleaf:
             leaf_etp = leaf_etp + cur.group_entropy * cur.group_size/dataset.shape[0] 
            else:

                if cur.left != None:
                 stack.append(cur.left)
                 
                if cur.right != None:
                 stack.append(cur.right)
        print(leaf_etp)
        
        
        
    def DFS_dotfile(self,TREE): #print the tree with dot format
        print('digraph Tree {node [shape=box, style="filled, rounded", color="black", fontname=helvetica] ;edge [fontname=helvetica] ;')

        stack = [TREE]

        while stack !=[]:
            cur = stack.pop()

            isleaf = (cur.left == None) & (cur.right == None)

            if isleaf:
                string = str(cur.node_id)+'[label=<node &#35;'+str(cur.node_id)+'<br/>entropy = '+str(round(cur.group_entropy,2))+'<br/>samples = '+str(cur.group_size)+' <br/>value = '+str(round(cur.bad_rate,3))+'>, fillcolor="#e58139fa"];' 
                string2 = str(cur.parent)+'->'+str(cur.node_id)+';'
                print(string)
                print(string2)
            else:

                string = str(cur.node_id)+'[label=<node &#35;'+str(cur.node_id)+'<br/>'+str(cur.feature)+'&le;'+str(cur.threshold)+'<br/>entropy = '+str(round(cur.group_entropy,2))+'<br/>samples = '+str(cur.group_size)+' <br/>value = '+str(round(cur.bad_rate,3))+'>, fillcolor="#e58139fa"];' 
                string2 = str(cur.parent)+'->'+str(cur.node_id)+';'
            
                print(string)
                print(string2)

            if cur.left != None:
                stack.append(cur.left)

            if cur.right != None:
                stack.append(cur.right)
        print('}')
                
    def DFS_auto_pruning(self,TREE,SIZE_LEAF,MAX_BR,DIFF_BR): #automatically prune the tree with parameters customized
         stack = [TREE]
         
         while stack !=[]:
             cur = stack.pop()
             
             isleaf = (cur.left == None) & (cur.right == None)
             
             if isleaf:
                 print('this is a leaf'+str(cur.node_id))
                 if cur.group_size < SIZE_LEAF:
                    cur = None

             elif cur.bad_rate >= MAX_BR:
                 cur.left = None
                 cur.right = None
                 print("tree is pruned at parent node: ",cur.node_id)      
                 
             elif cur.group_size < SIZE_LEAF:
                 cur.left = None
                 cur.right = None 
                 print("tree is pruned at parent node: ",cur.node_id)     
             elif abs(cur.left.bad_rate-cur.right.bad_rate) < DIFF_BR:
                 cur.left = None
                 cur.right = None
                 print("tree is pruned at parent node: ",cur.node_id)              
             elif (cur.left.group_size <SIZE_LEAF) or (cur.right.group_size <SIZE_LEAF):
                 cur.left = None
                 cur.right = None
                 print("tree is pruned at parent node: ",cur.node_id)                          
             else:         
                 if cur.left != None:
                     stack.append(cur.left)
                     
                 if cur.right != None:
                     stack.append(cur.right)            

    #取叶子的index，给原数据赋值prob
                     
    def bin_apply(self,data,feature,interval): #convert continuous data to binned data with given thresholds
        tmp = data[feature]
        binned_data = pd.cut(tmp,interval,duplicates  = 'drop',retbins = True)
        binned_data = df(binned_data[0])
        binned_data_labeled = binned_data[feature].cat.codes+1 
        
        return binned_data, binned_data_labeled   
        
        
    def piggyTree_diy(self, dataset,size_,thre_,size__,thre__):  #revise the split point manually
        
        node = TreeNode(dataset)  #TreeNode Value进行赋值
        node.bad_rate = dataset.rule104.mean()
        print(dataset.rule104.mean())
        
        node.group_entropy = self.entropy(dataset)
        node.group_size = dataset.shape[0]
        print(dataset.shape[0])
        

        if (dataset.shape[1] <= 1) | (dataset.shape[0] <= 0):
            return node
        
        if node.group_size == size_:
            feat = 'rule99'
            point = thre_
        elif node.group_size == size__:
            feat = 'rule91'
            point = thre__
        else:
            feat = self.find_best_feature(dataset)
            point = self.FindSplit(feat, dataset)
            
        node.feature = feat
        node.threshold = point
        
        left = df(dataset[dataset[feat] <= point])
        left.drop(feat,axis=1,inplace=True)
        right = df(dataset[dataset[feat] > point])
        right.drop(feat,axis=1,inplace=True)

        node.left = self.piggyTree_diy(left,size_,thre_,size__,thre__)
        node.right = self.piggyTree_diy(right,size_,thre_,size__,thre__)
        
        return node

class TreeNode(): #define attributes of each node
    def __init__(self,val,left = None , right = None ,parent = None, bad_rate = 0, group_entropy = 0, group_size = 0, feature = "", threshold = None, node_id = None):  #取等于=设置default value
        self.val = val
        self.left = left
        self.right = right
        self.bad_rate = bad_rate
        self.group_entropy = group_entropy
        self.group_size = group_size
        self.feature = feature
        self.threshold = threshold
        self.node_id = node_id
        self.parent = parent
