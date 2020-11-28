
# Deployment 

For deployment of AWS EKS Cluster you can follow the complete tutorial by clicking **[AWS EKS Deployment Tutorial](https://bit.ly/2BzEgYy)** or on the following **SoKube** image : 

[![SoKube](https://user-images.githubusercontent.com/58267422/95017213-6c5f3680-0658-11eb-8bda-9d8e5f986bcf.png)](https://bit.ly/2BzEgYy)

<br>

# Resources Destruction 

## Define all variables used in this tutorial 

```shell
export EKS_STACK_NAME="eks"
export EKS_AWS_REGION="us-east-1"
export EKS_KEY_PAIR_NAME="my-eks-key"
```

## Get Elastic Load Balancer (ELB) name

```shell
EKS_ELB_NAME=$(echo $EKS_ELB_HOSTNAME | awk -F "-" '{ print $1 }') \
&& echo $EKS_ELB_NAME
```

## ELB deletion

```shell
aws elb delete-load-balancer --region $EKS_AWS_REGION --load-balancer-name $EKS_ELB_NAME
```

## Get Security Group of ELB

```shell
EKS_ELB_SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
  --region $EKS_AWS_REGION \
  --filters Name=tag:kubernetes.io/cluster/EKS,Values=owned Name=group-name,Values=k8s-elb-$EKS_ELB_NAME \
  --query "SecurityGroups[*].{Name:GroupId}" \
  --output text) \
&& echo $EKS_ELB_SECURITY_GROUP_ID
```

## EKS cluster deletion with AWS CloudFormation stack

```shell
aws cloudformation delete-stack --region $EKS_AWS_REGION --stack-name $EKS_STACK_NAME
```

### Retrieve the EKS cluster deletion status  

```shell
started_date=$(date |  awk '{print $4}')
start=`date +%s`
while true; do 
  echo -e "EKS Cluster status : DELETE IN PROGRESS \n" 
  if [ "$(aws cloudformation describe-stacks --region $EKS_AWS_REGION --stack-name $EKS_STACK_NAME --query "Stacks[*].StackStatus" --output text)" == DELETE_FAILED ]; then
    echo -e "EKS Cluster status : DELETE FAILED \n"
    echo -e "You must delete the dependency object \n"
    end=`date +%s`
    runtime=$((end-start))
    finished_date=$(date |  awk '{print $4}')
    echo "started at :" $started_date 
    echo "finished at :" $finished_date
    hours=$((runtime / 3600)); minutes=$(( (runtime % 3600) / 60 )); seconds=$(( (runtime % 3600) % 60 )); echo "Total time : $hours h $minutes min $seconds sec"
    break
  fi
done
```

We should see : 

```shell
EKS Cluster status : DELETE IN PROGRESS

EKS Cluster status : DELETE IN PROGRESS

... 

EKS Cluster status : DELETE FAILED 

You must delete dependent objects

started at : 19:18:46
finished at : 19:46:49
Total time : 0 h 28 min 3 sec
```

> NB : The satck will not fully deleted beacause we have created ELB and we will have an interdependancy error with security groups. Follow the the next commandes 
to remove the stack entirely.


## ELB Security Groups deletion

```shell
aws ec2 delete-security-group --region $EKS_AWS_REGION --group-id $EKS_ELB_SECURITY_GROUP_ID
```

## EKS cluster deletion with AWS CloudFormation stack

```shell
aws cloudformation delete-stack --region $EKS_AWS_REGION --stack-name $EKS_STACK_NAME
```

> NB : This time the stack will be successfully deleted.

### Retrieve the EKS cluster deletion status

```shell
started_date=$(date |  awk '{print $4}')
start=`date +%s`
while true; do 
  echo -e "EKS Cluster status : DELETE IN PROGRESS \n" 
  aws cloudformation describe-stacks --region $EKS_AWS_REGION --stack-name $EKS_STACK_NAME --query "Stacks[*].StackStatus" --output text &>/dev/null
  if [[ "$?" -ne 0 ]]; then
    echo -e "EKS Cluster status : SUCCESSFULLY DELETED \n"
    end=`date +%s`
    runtime=$((end-start))
    finished_date=$(date |  awk '{print $4}')
    echo "started at :" $started_date 
    echo "finished at :" $finished_date
    hours=$((runtime / 3600)); minutes=$(( (runtime % 3600) / 60 )); seconds=$(( (runtime % 3600) % 60 )); echo "Total time : $hours h $minutes min $seconds sec"
    break
  fi
done
```

We should see :

```shell
EKS Cluster status : DELETE IN PROGRESS 

EKS Cluster status : DELETE IN PROGRESS 

...

EKS Cluster status : SUCCESSFULLY DELETED 

started at : 19:51:10
finished at : 19:51:20
Total time : 0 h 0 min 10 sec
```

## Key pair deletion

```shell
aws ec2 delete-key-pair --region $EKS_AWS_REGION --key-name $EKS_KEY_PAIR_NAME
```

## Private key file deletion

```shell
rm -rf $HOME/aws-eks/infrastructure-as-code/eks.id_rsa
```

## Kubeconfig file deletion

```shell
rm -fr $HOME/.kube/config
```

## aws-eks folder deletion

```shell
rm -fr $HOME/aws-eks
```