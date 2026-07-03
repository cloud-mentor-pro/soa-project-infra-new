## Tổng quan
File này trình bày lại cách build infra hiện tại
mục đích để AI đọc vào file này và hiểu được luồng hiện tại, tham khảo những resource cần thiết, từ đó viết lại dựa trên hướng dẫn mới

Folder lưu source infra tempalte hiện tại: /Users/phongnx/Main/01-PROJECT/SOA/SOA_PROJECT/soa-project-infra
Folder cần tham khảo để xây dựng lại tương tự: /Users/phongnx/Main/01-PROJECT/SOA/SOA_PROJECT/webapp-infra

## Guideline các bước cần thực hiện để build infra hiện tại
### 4.1 Dựng Network
Chạy stack: d-soa-network
Parameters: Giữ nguyên tất cả các parameter default	

### 4.2 Security group
Stack name: d-soa-security-groups
Parameters
    VpcId: soa-vpc-codeland-001

### 4.3 Tạo Database
Phần này tôi tạo thủ công bằng UI
Create Subnet group

1	Aurora and RDS -> Subnet groups -> Create SB subnet group					
						
2	Subnet group details					
		Name				soa-rds-subnet-group-codeland-database
		Description				subnet group for DB
		VPC				soa-vpc-codeland-001

3	Add subnets					
		Availability Zones: "us-east-1a", us-east-1c
		Subnets: soa-sn-codeland-private-db-01, soa-sn-codeland-private-db-02

Create Database
1	Aurora and RDS -> Databases -> Create database														
	Những setting không ghi trong thiết kế bên dưới thì cứ để default														

2	Engine options														
		Engine type			PostgreSQL										
		Engine version			PostgreSQL 17.x										


3	Templates				Free tier										

4	Settings														
		DB instance identifier: soa-rds-codeland-database								
		Master username: postgres								
		Credentials management: Managed in AWS Secrets Manager - most secure								

5	Instance configuration														
		Burstable classes (includes t classes): db.t3.micro						

6	Connectivity														
		Compute resource: Don’t connect to an EC2 compute resource						
		Network type: IPv4						
		Virtual private cloud (VPC): soa-vpc-codeland-001						
		DB subnet group: soa-rds-subnet-group-codeland-database						
		Public access: No						
		VPC security group (firewall): soa-sg-codeland-db						

7	Tags - optional														
		Key							Value						
		Name						soa-rds-codeland-database						
		Environment					develop						
		SystemID					SOA						
8	Monitoring
- Database insights - standard (checked)
- PostgresSQL log (checked)

9	Additional configuration
- Initial database name: dev
- Backup: Enable encryption, (default) aws/rds

### 4.4 Tạo Parameter store
Stack name				d-soa-parameter
Parameters				None

### 4.5 Tạo Compute resource (ECS + ALB + Target group + ECR + Role)
Stack name				d-soa-compute
Parameters							
		ALBSecurityGroupId			soa-sg-codeland-alb
		AppSecurityGroupId			soa-sg-codeland-ecs-task
		PrivateAppSubnet1Id			soa-sn-codeland-private-app-01
		PrivateAppSubnet2Id			soa-sn-codeland-private-app-02
		PublicSubnet1Id				soa-sn-codeland-public-01
		PublicSubnet2Id				soa-sn-codeland-public-02
		VPCId					    soa-vpc-codeland-001

### 4.5.1 Update giá trị trong parameter store
- Update value cho soa-param-codeland-db-secret-name: Copy giá trị Secret name đang liên kết với database
- Update value cho soa-param-codeland-db-url: Lấy thông tin Endpoint của database

### 4.6 Tạo server bastion
1. Giới thiệu
Bastion được connect thông qua Sesstion manager		
		
Thiết kế: Bastion sẽ được cài sẵn một số phần mềm và helper scripts		
	System Setup	
		Update hệ thống Amazon Linux
		Cài đặt và khởi động SSM Agent

	Database Tools	
		Cài đặt PostgreSQL client version 17
		Tạo script connect-db.sh để kết nối database từ Secrets Manager

	Development Tools	
		Cài đặt Git
		Cài đặt Node.js LTS + pnpm
		Cài đặt AWS CLI v2
		Cài đặt các tools: htop, tree, wget, curl, vim, nano, jq

	Container Platform	
		Cài đặt Docker + Docker Compose
		Add user ec2-user vào docker group

	Helper Scripts	
		connect-db.sh: Script kết nối PostgreSQL sử dụng credentials từ Secrets Manager
		get-soa-params.sh: Script lấy parameters từ Systems Manager

	Configuration	
		Tạo thư mục /home/ec2-user/projects
		Set quyền owner cho ec2-user
		Gửi signal hoàn thành tới CloudFormation stack

2. Create Stack bastion
Sử dụng Cloudformation -> Stack -> Create stack
Stack name: d-soa-bastion	
Parameters
		InstanceType		t3.micro
		KeyPairName			soa-keypair-codeland-bastion
		SecurityGroupId		soa-sg-codeland-bastion
		SubnetId			soa-sn-codeland-public-01
		VpcId				soa-vpc-codeland-001

3. Connect bastion và thực hiện một số công việc
Chi tiết ở bước tiếp theo

### 4.7 Connect bastion
1. Connect bastion thông qua SSM Section manager
2. Test kết nối đến DB
Run lệnh bên dưới
```
sudo su -
su - ec2-user
```

Test kết nối DB
```
./connect-db.sh
```

Thoát DB
```
\q
```

3. Migration DB
Access thư mục dự án 
cd /home/ec2-user/projects
Checkout backend
git clone {your-backend-repo-url} Ex: git clone https://github.com/{your-github-account}/soa-project-backend.git

cd soa-project-backend															

git checkout develop

Update thông tin kết nối DB trong: docker-compose-dev.yml	
- DB_PASSWORD={Cập nhật giá trị này bằng thông tin secrect manager}
- DB_URL={Cập nhật giá trị này bằng thông tin endpoint của rds}

Build backend để migrate DB
docker-compose -f docker-compose-dev.yml up -d --build

Kiểm tra Container đã UP
docker-compose -f docker-compose-dev.yml ps

4. Kiểm tra DB đã được Migration
Về thư mục gốc
```
cd
```

Connect DB
```
./connect-db.sh

-- Liệt kê databases
\l

-- Liệt kê tables
\dt
```

Kiểm tra data
SELECT * FROM users;

-- Thoát db
\q

CHÚC MỪNG BẠN
Như vậy là việc migration DB đã thành công

## 6. Config doamin, certificate
1. Tạo 2 Certificate
- api.{your domain or subdomain}
- {your domain or subdomain}

2. Update ALB để sử dụng Certificate
Add thêm Listener cho ALB

3. Tạo Route 53 Record Alias trỏ đến ALB
api.{your domain or subdomain} Alias -> trỏ đến ALB


## Công việc AI cần làm
Viết lại toàn bộ cách làm phía trên nhưng theo cách mới, tham khảo cách tổ chức của project: /Users/phongnx/Main/01-PROJECT/SOA/SOA_PROJECT/webapp-infra.
Bạn cần copy lại toàn bộ nội dung thư mục tham khảo.
Thêm, sửa, xoá các file cần thiết
Không nhất thiết tuân thủ theo các bước ở luồn build infra hiện tại, bạn hoàn toàn nghĩ ra cách làm mới, bao nhiêu stack, nhưng stack nào v.v
DB tôi đang build thủ công nhưng tôi cũng muốn tự động hoá bằng cloudformation luôn.
Luồng build hiện tại thủ công khá nhiều thứ nên bạn hãy suy nghĩ để công việc build dễ hiểu, thực hiện nhanh chóng, từng bước rõ ràng.

Tôi hy vọng bạn đã hiểu nội dung công việc, chổ nào cần confirm thì bạn cứ confirm với tôi nhé
