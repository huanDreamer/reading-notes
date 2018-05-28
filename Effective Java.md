# 对象的创建和销毁

### 一、使用静态工厂方法代替构造器

1. 静态工厂优势一：有名称。当一个类需要多个带有相同签名的构造器时，可以通过名称突出它们之间的区别

2. 静态工程优势二：不必每次都创建新的对象。可以将创建的实例缓存，或者是返回预先构建好的实例(工厂模式)。例如 `Boolean.valueOf(boolean)`。如果确保了每次返回的都是同一对象的话，可以使用`==`进行判断，提升效率

3. 静态工厂优势三：可以返回原返回类型的任何子类的对象，可以返回受保护的类的对象。例如`Collections.singletonLis(T)`、`EnumSet`。服务提供者框架：

   ```java
   public interface Service {
   	// Service-specific methods go here
   }
   
   public interface Provider {
   	Service newService();
   }
   
   public class Services {
   	private Services() {
   	} // Prevents instantiation (Item 4)
   
   	// Maps service names to services
   	private static final Map<String, Provider> providers = new ConcurrentHashMap<String, Provider>();
   	public static final String DEFAULT_PROVIDER_NAME = "<def>";
   
   	// Provider registration API
   	public static void registerDefaultProvider(Provider p) {
   		registerProvider(DEFAULT_PROVIDER_NAME, p);
   	}
   
   	public static void registerProvider(String name, Provider p) {
   		providers.put(name, p);
   	}
   
   	// Service access API
   	public static Service newInstance() {
   		return newInstance(DEFAULT_PROVIDER_NAME);
   	}
   
   	public static Service newInstance(String name) {
   		Provider p = providers.get(name);
   		if (p == null)
   			throw new IllegalArgumentException(
   					"No provider registered with name: " + name);
   		return p.newService();
   	}
   }
   
   public class Test {
   	public static void main(String[] args) {
   		// Providers would execute these lines
   		Services.registerDefaultProvider(DEFAULT_PROVIDER);
   		Services.registerProvider("comp", COMP_PROVIDER);
   		Services.registerProvider("armed", ARMED_PROVIDER);
   
   		// Clients would execute these lines
   		Service s1 = Services.newInstance();
   		Service s2 = Services.newInstance("comp");
   		Service s3 = Services.newInstance("armed");
   		System.out.printf("%s, %s, %s%n", s1, s2, s3);
   	}
   
   	private static Provider DEFAULT_PROVIDER = new Provider() {
   		public Service newService() {
   			return new Service() {
   				@Override
   				public String toString() {
   					return "Default service";
   				}
   			};
   		}
   	};
   
   	private static Provider COMP_PROVIDER = new Provider() {
   		public Service newService() {
   			return new Service() {
   				@Override
   				public String toString() {
   					return "Complementary service";
   				}
   			};
   		}
   	};
   
   	private static Provider ARMED_PROVIDER = new Provider() {
   		public Service newService() {
   			return new Service() {
   				@Override
   				public String toString() {
   					return "Armed service";
   				}
   			};
   		}
   	};
   }
   ```

   